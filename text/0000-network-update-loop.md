- Feature Name: `network-update-loop`
- Start Date: 2021-01-31
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

Often there is a need to update netcode systems like RPC queue, transport, and others outside the standard [`MonoBehaviour` event cycle](https://docs.unity3d.com/Manual/ExecutionOrder.html).

This RFC proposes a new **Network Update Loop** infrastructure that utilizes [Unity's low-level Player Loop API](https://docs.unity3d.com/ScriptReference/LowLevel.PlayerLoop.html) and allows for registering `INetworkUpdateSystem`s with `NetworkUpdate()` methods to be executed at specific `NetworkUpdateStage`s which may also be before or after `MonoBehaviour`-driven game logic execution.

Implementation is expected to have a minimal yet flexible API that would allow further systems such as network tick to be easily developed.

# Motivation
[motivation]: #motivation

Even though it is possible to use low-level Player Loop API directly to insert custom `PlayerLoopSystem`s into the existing player loop, it also requires a non-trivial amount of boilerplate code and could cause fundamental issues in the engine runtime if not done very carefully. **We would like to have less boilerplate code and also be less error-prone at runtime.**

Beyond that, the proposed design standardizes `NetworkUpdateStage`s which are going to be executed at specific points in the player loop. This allows other framework systems tied into `NetworkUpdateLoop` to be aligned, such as RPCs executing at specific stages. **We would like to standardize the execution of network stages as an infrastructure in the framework.**

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Developers will mostly interact with `NetworkUpdateLoop` for registration and `INetworkUpdateSystem` for implementation.

## NetworkUpdateLoop

`NetworkUpdateLoop` implements custom methods injected into the player loop, maintains stage-specific lists of `INetworkUpdateSystem`s.

### Registration

`NetworkUpdateLoop` exposes 4 methods for registration:

- `void RegisterNetworkUpdate(INetworkUpdateSystem updateSystem, NetworkUpdateStage updateStage)`
  - registers an `INetworkUpdateSystem` to be executed on the specified `NetworkUpdateStage`
- `void RegisterAllNetworkUpdates(INetworkUpdateSystem updateSystem)`
  - registers an `INetworkUpdateSystem` to be executed on all `NetworkUpdateStage`s
- `void UnregisterNetworkUpdate(INetworkUpdateSystem updateSystem, NetworkUpdateStage updateStage)`
  - unregisters an `INetworkUpdateSystem` from the specified `NetworkUpdateStage`
- `void UnregisterAllNetworkUpdates(INetworkUpdateSystem updateSystem)`
  - unregisters an `INetworkUpdateSystem` from all `NetworkUpdateStage`s

### Frames and Stages

After injection, player loop will look like this:

- **Initialization**
  - // other systems
  - RunNetworkInitialization
    - `UpdateStage = NetworkUpdateStage.Initialization`
    - `foreach (INetworkUpdateSystem in m_Initialization_Array)`
      - `INetworkUpdateSystem.NetworkUpdate()`
- **EarlyUpdate**
  - RunNetworkEarlyUpdate
    - `UpdateStage = NetworkUpdateStage.EarlyUpdate`
    - `foreach (INetworkUpdateSystem in m_EarlyUpdate_Array)`
      - `INetworkUpdateSystem.NetworkUpdate()`
  - // other systems
- **FixedUpdate**
  - RunNetworkFixedUpdate
    - `UpdateStage = NetworkUpdateStage.FixedUpdate`
    - `foreach (INetworkUpdateSystem in m_FixedUpdate_Array)`
      - `INetworkUpdateSystem.NetworkUpdate()`
  - ScriptRunBehaviourFixedUpdate
    - `foreach (MonoBehaviour in m_Behaviours)`
      - `MonoBehaviour.FixedUpdate()`
  - // other systems
- **PreUpdate**
  - RunNetworkPreUpdate
    - `UpdateStage = NetworkUpdateStage.PreUpdate`
    - `foreach (INetworkUpdateSystem in m_PreUpdate_Array)`
      - `INetworkUpdateSystem.NetworkUpdate()`
  - // other systems
- **Update**
  - RunNetworkUpdate
    - `UpdateStage = NetworkUpdateStage.Update`
    - `foreach (INetworkUpdateSystem in m_Update_Array)`
      - `INetworkUpdateSystem.NetworkUpdate()`
  - ScriptRunBehaviourUpdate
    - `foreach (MonoBehaviour in m_Behaviours)`
      - `MonoBehaviour.Update()`
  - // other systems
- **PreLateUpdate**
  - RunNetworkPreLateUpdate
    - `UpdateStage = NetworkUpdateStage.PreLateUpdate`
    - `foreach (INetworkUpdateSystem in m_PreLateUpdate_Array)`
      - `INetworkUpdateSystem.NetworkUpdate()`
  - // other systems
  - ScriptRunBehaviourLateUpdate
    - `foreach (MonoBehaviour in m_Behaviours)`
      - `MonoBehaviour.LateUpdate()`
- **PostLateUpdate**
  - // other systems
  - RunNetworkPostLateUpdate
    - `UpdateStage = NetworkUpdateStage.PostLateUpdate`
    - `foreach (INetworkUpdateSystem in m_PostLateUpdate_Array)`
      - `INetworkUpdateSystem.NetworkUpdate()`

As seen above, player loop executes `Initialization` stage and that invokes `NetworkUpdateLoop`'s `RunNetworkInitialization` method which iterates over registered `INetworkUpdateSystem`s in `m_Initialization_Array` and calls `INetworkUpdateSystem.NetworkUpdate()` on them.

In all stages, iterate over a static array and call `NetworkUpdate()` method over `INetworkUpdateSystem` interface pattern is repeated.

## INetworkUpdateSystem

`INetworkUpdateSystem` interface is the ultimate contract between user-defined systems and the `NetworkUpdateLoop`.

### Plain Class Usage

```cs
public class MyPlainScript : IDisposable, INetworkUpdateSystem
{
    public void Initialize()
    {
        // extension method calls are equivalent to `NetworkUpdateLoop.RegisterNetworkUpdate(this, NetworkUpdateStage.XXX)`
        this.RegisterNetworkUpdate(NetworkUpdateStage.EarlyUpdate);
        this.RegisterNetworkUpdate(NetworkUpdateStage.FixedUpdate);
        this.RegisterNetworkUpdate(NetworkUpdateStage.Update);
    }

    // will be called on EarlyUpdate, FixedUpdate and Update stages as registered above
    public void NetworkUpdate()
    {
        Debug.Log($"{nameof(MyPlainScript)}.{nameof(NetworkUpdate)}({NetworkUpdateLoop.UpdateStage})");
    }

    public void Dispose()
    {
        // extension method call is equivalent to `NetworkUpdateLoop.UnregisterAllNetworkUpdates(this)`
        this.UnregisterAllNetworkUpdates();
    }
}
```

Output

```
MyPlainScript.NetworkUpdate(EarlyUpdate)
MyPlainScript.NetworkUpdate(FixedUpdate)
MyPlainScript.NetworkUpdate(Update)
MyPlainScript.NetworkUpdate(EarlyUpdate)
MyPlainScript.NetworkUpdate(FixedUpdate)
MyPlainScript.NetworkUpdate(Update)
...
```

### MonoBehaviour Usage

```cs
public class MyGameScript : MonoBehaviour, INetworkUpdateSystem
{
    private void OnEnable()
    {
        // extension method calls are equivalent to `NetworkUpdateLoop.RegisterNetworkUpdate(this, NetworkUpdateStage.XXX)`
        this.RegisterNetworkUpdate(NetworkUpdateStage.EarlyUpdate);
        this.RegisterNetworkUpdate(NetworkUpdateStage.FixedUpdate);
        this.RegisterNetworkUpdate(NetworkUpdateStage.Update);
    }

    // will be called on EarlyUpdate, FixedUpdate and Update stages as registered above
    public void NetworkUpdate()
    {
        Debug.Log($"{nameof(MyGameScript)}.{nameof(NetworkUpdate)}({NetworkUpdateLoop.UpdateStage})");
    }

    private void FixedUpdate()
    {
        Debug.Log($"{nameof(MyGameScript)}.{nameof(FixedUpdate)}()");
    }

    private void Update()
    {
        Debug.Log($"{nameof(MyGameScript)}.{nameof(Update)}()");
    }

    private void OnDisable()
    {
        // extension method call is equivalent to `NetworkUpdateLoop.UnregisterAllNetworkUpdates(this)`
        this.UnregisterAllNetworkUpdates();
    }
}
```

Output

```
MyGameScript.NetworkUpdate(EarlyUpdate)
MyGameScript.NetworkUpdate(FixedUpdate)
MyGameScript.FixedUpdate()
MyGameScript.NetworkUpdate(Update)
MyGameScript.Update()
MyGameScript.NetworkUpdate(EarlyUpdate)
MyGameScript.NetworkUpdate(FixedUpdate)
MyGameScript.FixedUpdate()
MyGameScript.NetworkUpdate(Update)
MyGameScript.Update()
...
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Injecting NetworkUpdateLoop Systems Into PlayerLoop

![](https://www.plantuml.com/plantuml/svg/tLbBSzis4BxxL_2OpDIP99DBCsrdP4bMrKg9Ayg9RncON8aOWm0BWBRPNzy5I4dK4qlv5Fia0xiBO7U_VT4yS5ampV8ejfM0o5ZD4rlcuiAiTt8bLP9fbKDaTGcjdHGMh1JsMaaRy_yW-l3fua8g2v5w11tT25Q1dYXLV8vk9RTLzF7zXr_VVOecyxX5zWNiaz8FjqNABDJKir9QdiCOcJWJAFchd2YMJi5cAhtOOuWk_pWaPSD-cALzln7OVgarIBjPFS3rs-LbT98WS1DckPBlUcdwfg7QtdbBbZXaMbJrmhknDA3jc_U_h0XRctI9r_mPqcCIMvjqYJSQZibRRz6b7HDrngqc-C-9csssEqXLnhR4jYoylVCsx8enTFB1fLPbiUDX5n7-RMXsJwX_roBdYBC0-O1hIApICSiBGM7ecKecjuL0o3hgMF1avJq1pXWPCT98nEnO5gKrTFhOXiNVpLhGE41xE1GTXAhdxkQxUCCbHByVifLmlFcFxPqVFQrPvVUF7ZRUMeqhOo7_l4rwGe1pfKCIF0l8T5T5ZdvFoeKZhBtm1ThTPJrFjRjI6hssbdyMfGOGh8Ie_-yhAH6T1D1hjU02DAfAsslCtY6mZ8hIeTTOX7q8y9A9olB4z9NsG3GajFkYu3lSp-3V4iyXyGoV8IajJ1cNyqm3I_zIzoRyz0ICcQhIO5ZvpcMgdW8iVG5CZwK5dKFAyLepj1neq8R2EE4ImfS2W-N65IpKAXa6E6R8K5lEgxYtmDirKaAAzk9zM9wXokvPyZ1b4Y6ZIOonHKNdN2wxCCtXsPRwF5vybKiUo1LKIMsLByhoHJMMZzpmUm6a_ESi9SkYpv8CqW6pZ7GrvtWZDC4t3ayp11w9gS_a8pEEVAwPD6ZnF9YfQ0ZoH3vLqlJRG646kMhuGXasqRom6AS7u5PJ7pxAWX90wilzpMLOJIYSOoymLWPNhTFFy17TEeIXEPDBHLBKW9k2sIJpOlyfeGJI0JTPPJ2iUUJvQ50el1-aQpDYTSS9mJSkD4S0Mi5COoVIhMz0kqEOJ60TOXl6JT9jrUPTa-ukwtb9TrTAatHRowyZwP93-nTuyj0wRPb39RRdeDVPPSeL_P0rbAyWOuzSbVhC2v8u9h1aIUNfUvBdlg8bCTxDw8Sq-EpCy77GcQKlbCrUf33N2JR2EWPTByI7tDaszczEb0udaR_DU1xkjsKqg5yllf92kxPNDzyYKVSpI1wmIBhc859A9XvFHxVZ6D_bqrTAob-hI4IB3oZRryWaqRqnIQYhjZkKCjzu2ghbuETvzujTUz-lNvDJxoU2RlcZYKg3Ic9bdspYKZVsEJCFXXX5zugUhJ7T6S5lX_di0BWZPnC4DnklmsTMS7T9hBPa9zEjIu0hM799AXswLGr7ZXlkl_u8um0MeBMBgqoKX3c4J4UQ5Ha6V3U1yCruhHTQz5VIM8szfEtK3s3lnHFHcDUhidcl49qnttBT5gwsfjMaFjgbZTLKGcDlHDGt-PGNXfqcAySFN4f8VHNNKXmfANLftDkpzRhj3xOBZGbTCH-W_7PzTuQV35SPEEmbnxsQRAxcwQMnAg_7CyqaYAW68QYZjg4KV9bPaCtdmWxNmrCQXvY2oO0uGFmEzxv6mfy8sM0Hhz08SZbok39TIw1EEgkB9XhiQkMEStm0oPE7ERUr7uz0vM1rrHrlrrnoav5KK9FMsFOt3Sy0hznSPwhIis9MNxbUc-hKhF0ZpKG3I9CfInlnUXGDDAFDdFEE0-UT1_aciyDGU0P6YP8mml9znB0v9JINqto9EGsflovgDJCXtDGvz1ZiSBmMI1r8d37Kq9zCfWA6MgkWgfen7hwZzyo3iqN_TG7IIJs87i1zkQH-Ojd-AXawQGgvqfNNavikOq1p0p_QTn0tYkw3cqnILRowv0bJdYwUzok4CwldVK6hydbhvoVw3Fe_)

<details>
  <summary>UML Source</summary>

```
skinparam Style strictuml
skinparam monochrome true
skinparam defaultFontSize 14

note over PlayerLoop: Unity 2019.4 LTS
note over NetworkUpdateLoop: RuntimeInitializeOnLoadMethod
NetworkUpdateLoop -> NetworkUpdateLoop: Initialize
NetworkUpdateLoop -> PlayerLoop: GetCurrentPlayerLoop
NetworkUpdateLoop <-- PlayerLoop
NetworkUpdateLoop -> NetworkUpdateLoop: Initialization.Add(NetworkInitialization)
NetworkUpdateLoop -> NetworkUpdateLoop: EarlyUpdate.Insert(0, NetworkEarlyUpdate)
NetworkUpdateLoop -> NetworkUpdateLoop: FixedUpdate.Insert(0, NetworkFixedUpdate)
NetworkUpdateLoop -> NetworkUpdateLoop: PreUpdate.Insert(0, NetworkPreUpdate)
NetworkUpdateLoop -> NetworkUpdateLoop: Update.Insert(0, NetworkUpdate)
NetworkUpdateLoop -> NetworkUpdateLoop: PreLateUpdate.Insert(0, NetworkPreLateUpdate)
NetworkUpdateLoop -> NetworkUpdateLoop: PostLateUpdate.Add(NetworkPostLateUpdate)
NetworkUpdateLoop -> PlayerLoop: SetPlayerLoop
NetworkUpdateLoop <-- PlayerLoop
group Initialization
    PlayerLoop -> PlayerLoop: PlayerUpdateTime
    PlayerLoop -> PlayerLoop: DirectorSampleTime
    PlayerLoop -> PlayerLoop: AsyncUploadTimeSlicedUpdate
    PlayerLoop -> PlayerLoop: SynchronizeInputs
    PlayerLoop -> PlayerLoop: SynchronizeState
    PlayerLoop -> PlayerLoop: XREarlyUpdate
    PlayerLoop -> NetworkUpdateLoop: RunNetworkInitialization
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = Initialization
    loop m_Initialization_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
end
group EarlyUpdate
    PlayerLoop -> NetworkUpdateLoop: RunNetworkEarlyUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = EarlyUpdate
    loop m_EarlyUpdate_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
    PlayerLoop -> PlayerLoop: PollPlayerConnection
    PlayerLoop -> PlayerLoop: ProfilerStartFrame
    PlayerLoop -> PlayerLoop: GpuTimestamp
    PlayerLoop -> PlayerLoop: AnalyticsCoreStatsUpdate
    PlayerLoop -> PlayerLoop: UnityWebRequestUpdate
    PlayerLoop -> PlayerLoop: ExecuteMainThreadJobs
    PlayerLoop -> PlayerLoop: ProcessMouseInWindow
    PlayerLoop -> PlayerLoop: ClearIntermediateRenderers
    PlayerLoop -> PlayerLoop: ClearLines
    PlayerLoop -> PlayerLoop: PresentBeforeUpdate
    PlayerLoop -> PlayerLoop: ResetFrameStatsAfterPresent
    PlayerLoop -> PlayerLoop: UpdateAsyncReadbackManager
    PlayerLoop -> PlayerLoop: UpdateStreamingManager
    PlayerLoop -> PlayerLoop: UpdateTextureStreamingManager
    PlayerLoop -> PlayerLoop: UpdatePreloading
    PlayerLoop -> PlayerLoop: RendererNotifyInvisible
    PlayerLoop -> PlayerLoop: PlayerCleanupCachedData
    PlayerLoop -> PlayerLoop: UpdateMainGameViewRect
    PlayerLoop -> PlayerLoop: UpdateCanvasRectTransform
    PlayerLoop -> PlayerLoop: XRUpdate
    PlayerLoop -> PlayerLoop: UpdateInputManager
    PlayerLoop -> PlayerLoop: ProcessRemoteInput
    PlayerLoop -> PlayerLoop: ScriptRunDelayedStartupFrame
    PlayerLoop -> PlayerLoop: UpdateKinect
    PlayerLoop -> PlayerLoop: DeliverIosPlatformEvents
    PlayerLoop -> PlayerLoop: TangoUpdate
    PlayerLoop -> PlayerLoop: DispatchEventQueueEvents
    PlayerLoop -> PlayerLoop: PhysicsResetInterpolatedTransformPosition
    PlayerLoop -> PlayerLoop: SpriteAtlasManagerUpdate
    PlayerLoop -> PlayerLoop: PerformanceAnalyticsUpdate
end
group FixedUpdate
    PlayerLoop -> NetworkUpdateLoop: RunNetworkFixedUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = FixedUpdate
    loop m_FixedUpdate_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
    PlayerLoop -> PlayerLoop: ClearLines
    PlayerLoop -> PlayerLoop: NewInputFixedUpdate
    PlayerLoop -> PlayerLoop: DirectorFixedSampleTime
    PlayerLoop -> PlayerLoop: AudioFixedUpdate
    PlayerLoop -> PlayerLoop: ScriptRunBehaviourFixedUpdate
    PlayerLoop -> PlayerLoop: DirectorFixedUpdate
    PlayerLoop -> PlayerLoop: LegacyFixedAnimationUpdate
    PlayerLoop -> PlayerLoop: XRFixedUpdate
    PlayerLoop -> PlayerLoop: PhysicsFixedUpdate
    PlayerLoop -> PlayerLoop: Physics2DFixedUpdate
    PlayerLoop -> PlayerLoop: PhysicsClothFixedUpdate
    PlayerLoop -> PlayerLoop: DirectorFixedUpdatePostPhysics
    PlayerLoop -> PlayerLoop: ScriptRunDelayedFixedFrameRate
end
group PreUpdate
    PlayerLoop -> NetworkUpdateLoop: RunNetworkPreUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = PreUpdate
    loop m_PreUpdate_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
    PlayerLoop -> PlayerLoop: PhysicsUpdate
    PlayerLoop -> PlayerLoop: Physics2DUpdate
    PlayerLoop -> PlayerLoop: CheckTexFieldInput
    PlayerLoop -> PlayerLoop: IMGUISendQueuedEvents
    PlayerLoop -> PlayerLoop: NewInputUpdate
    PlayerLoop -> PlayerLoop: SendMouseEvents
    PlayerLoop -> PlayerLoop: AIUpdate
    PlayerLoop -> PlayerLoop: WindUpdate
    PlayerLoop -> PlayerLoop: UpdateVideo
end
group Update
    PlayerLoop -> NetworkUpdateLoop: RunNetworkUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = Update
    loop m_Update_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
    PlayerLoop -> PlayerLoop: ScriptRunBehaviourUpdate
    PlayerLoop -> PlayerLoop: ScriptRunDelayedDynamicFrameRate
    PlayerLoop -> PlayerLoop: ScriptRunDelayedTasks
    PlayerLoop -> PlayerLoop: DirectorUpdate
end
group PreLateUpdate
    PlayerLoop -> NetworkUpdateLoop: RunNetworkPreLateUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = PreLateUpdate
    loop m_PreLateUpdate_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
    PlayerLoop -> PlayerLoop: AIUpdatePostScript
    PlayerLoop -> PlayerLoop: DirectorUpdateAnimationBegin
    PlayerLoop -> PlayerLoop: LegacyAnimationUpdate
    PlayerLoop -> PlayerLoop: DirectorUpdateAnimationEnd
    PlayerLoop -> PlayerLoop: DirectorDeferredEvaluate
    PlayerLoop -> PlayerLoop: EndGraphicsJobsAfterScriptUpdate
    PlayerLoop -> PlayerLoop: ConstraintManagerUpdate
    PlayerLoop -> PlayerLoop: ParticleSystemBeginUpdateAll
    PlayerLoop -> PlayerLoop: ScriptRunBehaviourLateUpdate
end
group PostLateUpdate
    PlayerLoop -> PlayerLoop: PlayerSendFrameStarted
    PlayerLoop -> PlayerLoop: DirectorLateUpdate
    PlayerLoop -> PlayerLoop: ScriptRunDelayedDynamicFrameRate
    PlayerLoop -> PlayerLoop: PhysicsSkinnedClothBeginUpdate
    PlayerLoop -> PlayerLoop: UpdateRectTransform
    PlayerLoop -> PlayerLoop: PlayerUpdateCanvases
    PlayerLoop -> PlayerLoop: UpdateAudio
    PlayerLoop -> PlayerLoop: VFXUpdate
    PlayerLoop -> PlayerLoop: ParticleSystemEndUpdateAll
    PlayerLoop -> PlayerLoop: EndGraphicsJobsAfterScriptLateUpdate
    PlayerLoop -> PlayerLoop: UpdateCustomRenderTextures
    PlayerLoop -> PlayerLoop: UpdateAllRenderers
    PlayerLoop -> PlayerLoop: EnlightenRuntimeUpdate
    PlayerLoop -> PlayerLoop: UpdateAllSkinnedMeshes
    PlayerLoop -> PlayerLoop: ProcessWebSendMessages
    PlayerLoop -> PlayerLoop: SortingGroupsUpdate
    PlayerLoop -> PlayerLoop: UpdateVideoTextures
    PlayerLoop -> PlayerLoop: UpdateVideo
    PlayerLoop -> PlayerLoop: DirectorRenderImage
    PlayerLoop -> PlayerLoop: PlayerEmitCanvasGeometry
    PlayerLoop -> PlayerLoop: PhysicsSkinnedClothFinishUpdate
    PlayerLoop -> PlayerLoop: FinishFrameRendering
    PlayerLoop -> PlayerLoop: BatchModeUpdate
    PlayerLoop -> PlayerLoop: PlayerSendFrameComplete
    PlayerLoop -> PlayerLoop: UpdateCaptureScreenshot
    PlayerLoop -> PlayerLoop: PresentAfterDraw
    PlayerLoop -> PlayerLoop: ClearImmediateRenderers
    PlayerLoop -> PlayerLoop: PlayerSendFramePostPresent
    PlayerLoop -> PlayerLoop: UpdateResolution
    PlayerLoop -> PlayerLoop: InputEndFrame
    PlayerLoop -> PlayerLoop: TriggerEndOfFrameCallbacks
    PlayerLoop -> PlayerLoop: GUIClearEvents
    PlayerLoop -> PlayerLoop: ShaderHandleErrors
    PlayerLoop -> PlayerLoop: ResetInputAxis
    PlayerLoop -> PlayerLoop: ThreadedLoadingDebug
    PlayerLoop -> PlayerLoop: ProfilerSynchronizeStats
    PlayerLoop -> PlayerLoop: MemoryFrameMaintenance
    PlayerLoop -> PlayerLoop: ExecuteGameCenterCallbacks
    PlayerLoop -> PlayerLoop: ProfilerEndFrame
    PlayerLoop -> NetworkUpdateLoop: RunNetworkPostLateUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = PostLateUpdate
    loop m_PostLateUpdate_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
end
```
</details>

### Pseudo-code

```cs
public interface INetworkUpdateSystem
{
    void NetworkUpdate();
}

public enum NetworkUpdateStage
{
    Initialization = -4,
    EarlyUpdate = -3,
    FixedUpdate = -2,
    PreUpdate = -1,
    Update = 0,
    PreLateUpdate = 1,
    PostLateUpdate = 2
}

public static class NetworkUpdateLoop
{
    [RuntimeInitializeOnLoadMethod]
    private static void Initialize()
    {
        var customPlayerLoop = PlayerLoop.GetCurrentPlayerLoop();

        for (int i = 0; i < customPlayerLoop.subSystemList.Length; i++)
        {
            var playerLoopSystem = customPlayerLoop.subSystemList[i];

            if (playerLoopSystem.type == typeof(Initialization))
            {
                var subsystems = playerLoopSystem.subSystemList.ToList();
                subsystems.Add(NetworkInitialization.CreateLoopSystem());
                playerLoopSystem.subSystemList = subsystems.ToArray();
            }
            else if (playerLoopSystem.type == typeof(EarlyUpdate))
            {
                var subsystems = playerLoopSystem.subSystemList.ToList();
                subsystems.Insert(0, NetworkEarlyUpdate.CreateLoopSystem());
                playerLoopSystem.subSystemList = subsystems.ToArray();
            }
            // add/insert other loop systems ...

            customPlayerLoop.subSystemList[i] = playerLoopSystem;
        }

        PlayerLoop.SetPlayerLoop(customPlayerLoop);
    }

    private struct NetworkInitialization
    {
        public static PlayerLoopSystem CreateLoopSystem()
        {
            return new PlayerLoopSystem
            {
                type = typeof(NetworkInitialization),
                updateDelegate = RunNetworkInitialization
            };
        }
    }

    private struct NetworkEarlyUpdate
    {
        public static PlayerLoopSystem CreateLoopSystem()
        {
            return new PlayerLoopSystem
            {
                type = typeof(NetworkEarlyUpdate),
                updateDelegate = RunNetworkEarlyUpdate
            };
        }
    }

    // define other loop systems ...

    public static NetworkUpdateStage UpdateStage;

    private static readonly List<INetworkUpdateSystem> m_Initialization_List = new List<INetworkUpdateSystem>();
    private static INetworkUpdateSystem[] m_Initialization_Array = new INetworkUpdateSystem[0];

    private static void RunNetworkInitialization()
    {
        UpdateStage = NetworkUpdateStage.Initialization;
        int arrayLength = m_Initialization_Array.Length;
        for (int i = 0; i < arrayLength; i++)
        {
            m_Initialization_Array[i].NetworkUpdate();
        }
    }

    private static readonly List<INetworkUpdateSystem> m_EarlyUpdate_List = new List<INetworkUpdateSystem>();
    private static INetworkUpdateSystem[] m_EarlyUpdate_Array = new INetworkUpdateSystem[0];

    private static void RunNetworkEarlyUpdate()
    {
        UpdateStage = NetworkUpdateStage.EarlyUpdate;
        int arrayLength = m_EarlyUpdate_Array.Length;
        for (int i = 0; i < arrayLength; i++)
        {
            m_EarlyUpdate_Array[i].NetworkUpdate();
        }
    }

    // implement other loop systems ...

    public static void RegisterNetworkUpdate(this INetworkUpdateSystem updateSystem, NetworkUpdateStage updateStage = NetworkUpdateStage.Update)
    {
        switch (updateStage)
        {
            case NetworkUpdateStage.Initialization:
            {
                if (!m_Initialization_List.Contains(updateSystem))
                {
                    m_Initialization_List.Add(updateSystem);
                    m_Initialization_Array = m_Initialization_List.ToArray();
                }

                break;
            }
            case NetworkUpdateStage.EarlyUpdate:
            {
                if (!m_EarlyUpdate_List.Contains(updateSystem))
                {
                    m_EarlyUpdate_List.Add(updateSystem);
                    m_EarlyUpdate_Array = m_EarlyUpdate_List.ToArray();
                }

                break;
            }
            // register other loop systems ...
        }
    }

    public static void UnregisterNetworkUpdate(this INetworkUpdateSystem updateSystem, NetworkUpdateStage updateStage = NetworkUpdateStage.Update)
    {
        switch (updateStage)
        {
            case NetworkUpdateStage.Initialization:
            {
                if (m_Initialization_List.Contains(updateSystem))
                {
                    m_Initialization_List.Remove(updateSystem);
                    m_Initialization_Array = m_Initialization_List.ToArray();
                }

                break;
            }
            case NetworkUpdateStage.EarlyUpdate:
            {
                if (m_EarlyUpdate_List.Contains(updateSystem))
                {
                    m_EarlyUpdate_List.Remove(updateSystem);
                    m_EarlyUpdate_Array = m_EarlyUpdate_List.ToArray();
                }

                break;
            }
            // unregister other loop systems ...
        }
    }
}
```

## NetworkUpdateLoop Running INetworkUpdateSystem Updates

![](https://www.plantuml.com/plantuml/svg/xLPTJy8m57tdL_HH7nWnyOce2KOm9AB6XMTIsHKQkdRfEdJ-UlS5Qxjki16DYV8qUkyzfxtdxAxXXh002-mZLyOKK2W5MSh8fxrm7_4vuykru3uWAI9G8XwyuOZA2MVI9P-0BYvxFSOb8Bu5WMRnCyM4kKj10Zb4qpiI1Zp4hnGQaXv1ldEncGSUbk36eGHVoxx7FkoIPyd6Rc6DjuH7eZRB2haIF0fqScVAY2IO9WVfeUId1L7_1cau3vm7G_G2AvBW2IrqDiQ2nldpkGNggj-lOfr8EI4VuFqiPLisODwkxQfkpXCRiymKEL0ftQazLv2Qpj-HqDBnxoLioLMsEv4c1f4kEagNCfmoLBULY1MhPcabkGQXUEqaNW6wHYOAJGlzXRAy60c1uonOIsCC3IsdeJ9jb5PwY4KT8-r8oieiDHN3w7UzGtHHodz3D1WWnt7iqYf-R2kjMTfDMXEba5PP_YlIsbLhJieH4qtsWz7ifsrscZbL3lsajdnp8EaLokOj1kxU3Qk7kzdtPETMJVi_YeBMVZrWrHOk_MK6DQyhoJsspNrbpaJnFHzHgiN3zjzovHBjv8_7NrOFR-IeIzmN)

<details>
  <summary>UML Source</summary>

```
skinparam Style strictuml
skinparam monochrome true
skinparam defaultFontSize 14

note over MyPlainScript: IDisposable
note over MyPlainScript: INetworkUpdateSystem
note over MyGameScript: MonoBehaviour
note over MyGameScript: INetworkUpdateSystem
group MyPlainScript.Initialize
    MyPlainScript -> NetworkUpdateLoop: RegisterNetworkUpdate(EarlyUpdate)
    MyPlainScript <-- NetworkUpdateLoop
    MyPlainScript -> NetworkUpdateLoop: RegisterNetworkUpdate(FixedUpdate)
    MyPlainScript <-- NetworkUpdateLoop
    MyPlainScript -> NetworkUpdateLoop: RegisterNetworkUpdate(Update)
    MyPlainScript <-- NetworkUpdateLoop
end
group MonoBehaviour.OnEnable
    MyGameScript -> NetworkUpdateLoop: RegisterNetworkUpdate(EarlyUpdate)
    MyGameScript <-- NetworkUpdateLoop
    MyGameScript -> NetworkUpdateLoop: RegisterNetworkUpdate(FixedUpdate)
    MyGameScript <-- NetworkUpdateLoop
    MyGameScript -> NetworkUpdateLoop: RegisterNetworkUpdate(Update)
    MyGameScript <-- NetworkUpdateLoop
end
group PlayerLoop.EarlyUpdate
    PlayerLoop -> NetworkUpdateLoop: RunNetworkEarlyUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = EarlyUpdate
    loop m_EarlyUpdate_Array
        NetworkUpdateLoop -> MyPlainScript: NetworkUpdate
        NetworkUpdateLoop <-- MyPlainScript
        NetworkUpdateLoop -> MyGameScript: NetworkUpdate
        NetworkUpdateLoop <-- MyGameScript
    end
    PlayerLoop <-- NetworkUpdateLoop
    PlayerLoop -> PlayerLoop: // ...
end
group PlayerLoop.FixedUpdate
    PlayerLoop -> NetworkUpdateLoop: RunNetworkFixedUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = FixedUpdate
    loop m_FixedUpdate_Array
        NetworkUpdateLoop -> MyPlainScript: NetworkUpdate
        NetworkUpdateLoop <-- MyPlainScript
        NetworkUpdateLoop -> MyGameScript: NetworkUpdate
        NetworkUpdateLoop <-- MyGameScript
    end
    PlayerLoop -> PlayerLoop: // ...
    PlayerLoop -> PlayerLoop: ScriptRunBehaviourFixedUpdate
    group MonoBehaviour.FixedUpdate
        PlayerLoop -> MyGameScript: FixedUpdate
        MyGameScript -> MyGameScript: // ...
        PlayerLoop <-- MyGameScript
    end
    PlayerLoop -> PlayerLoop: // ...
end
group PlayerLoop.Update
    PlayerLoop -> NetworkUpdateLoop: RunNetworkUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = Update
    loop m_Update_Array
        NetworkUpdateLoop -> MyPlainScript: NetworkUpdate
        NetworkUpdateLoop <-- MyPlainScript
        NetworkUpdateLoop -> MyGameScript: NetworkUpdate
        NetworkUpdateLoop <-- MyGameScript
    end
    PlayerLoop <-- NetworkUpdateLoop
    PlayerLoop -> PlayerLoop: ScriptRunBehaviourUpdate
    group MonoBehaviour.Update
        PlayerLoop -> MyGameScript: Update
        MyGameScript -> MyGameScript: // ...
        PlayerLoop <-- MyGameScript
    end
    PlayerLoop -> PlayerLoop: // ...
end
group MonoBehaviour.OnDisable
    MyGameScript -> NetworkUpdateLoop: UnregisterAllNetworkUpdates
    MyGameScript <-- NetworkUpdateLoop
end
group IDisposable.Dispose
    MyPlainScript -> NetworkUpdateLoop: UnregisterAllNetworkUpdates
    MyPlainScript <-- NetworkUpdateLoop
end
```
</details>

# Drawbacks
[drawbacks]: #drawbacks

N/A: We need to have this, other systems such as network tick, network variable snapshotting and others will be relying on this pipeline.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
  - This approach is both more flexible and performant than other approaches considered (manual centralized UpdateManager, [Script Execution Order](https://docs.unity3d.com/Manual/class-MonoManager.html) etc.)
- What other designs have been considered and what is the rationale for not choosing them?
  - We might rely on [Script Execution Order](https://docs.unity3d.com/Manual/class-MonoManager.html) which lives in [`MonoBehaviour` event cycle](https://docs.unity3d.com/Manual/ExecutionOrder.html) but direct access and control over player loop is better in terms of performance and flexibility.
- What is the impact of not doing this?
  - Other upcoming network systems are depending on this feature.

# Prior art
[prior-art]: #prior-art

N/A

# Unresolved questions
[unresolved-questions]: #unresolved-questions

N/A

# Future possibilities
[future-possibilities]: #future-possibilities

## Network Tick & Time API

Relying on this pipeline, we could implement network time and tick numbers to reference specific network frames — potentially for interpolation, snapshotting, rollback/replay etc.

## Transport IO

A transport system could be polled very early in the `EarlyUpdate` stage to receive input data and could be flushed on `PostLateUpdate` stage to send output data.

## RPC Execution

RPCs might carry `NetworkUpdateStage` information with them and be executed at specific stages on the remote-end.

## Network Variables Snapshotting

Network variables need to know when they should be written and when they should be read so that they could snapshot themselves for outbound and update themselves for inbound.

## Other Tick/Update Dependent Network Systems

With this network update loop pipeline, it's natural to register any tick/frame/update based network systems into relevant stages. We are expecting lots of upcoming network systems to utilize this infrastructure in the future.
