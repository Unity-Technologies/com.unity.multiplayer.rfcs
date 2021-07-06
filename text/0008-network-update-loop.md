# Network Update Loop
[feature]: #feature

- Start Date: `2021-01-31`
- RFC PR: [#8](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/8)
- SDK PR: [#479](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/pull/479)

# Summary
[summary]: #summary

Often there is a need to update netcode systems like RPC queue, transport, and others outside the standard [`MonoBehaviour` event cycle](https://docs.unity3d.com/Manual/ExecutionOrder.html).

This RFC proposes a new **Network Update Loop** infrastructure that utilizes [Unity's low-level Player Loop API](https://docs.unity3d.com/ScriptReference/LowLevel.PlayerLoop.html) and allows for registering `INetworkUpdateSystem`s with `NetworkUpdate` methods to be executed at specific `NetworkUpdateStage`s which may also be before or after `MonoBehaviour`-driven game logic execution.

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

### Update Stages

After injection, player loop will look like this:

- **Initialization**
  - // other systems
  - RunNetworkInitialization
    - `UpdateStage = NetworkUpdateStage.Initialization`
    - `foreach (INetworkUpdateSystem in m_Initialization_Array)`
      - `INetworkUpdateSystem.NetworkUpdate(UpdateStage)`
- **EarlyUpdate**
  - // other systems
  - RunNetworkEarlyUpdate
    - `UpdateStage = NetworkUpdateStage.EarlyUpdate`
    - `foreach (INetworkUpdateSystem in m_EarlyUpdate_Array)`
      - `INetworkUpdateSystem.NetworkUpdate(UpdateStage)`
  - ScriptRunDelayedStartupFrame
    - `foreach (MonoBehaviour in m_Behaviours)`
      - `MonoBehaviour.Awake()`
      - `MonoBehaviour.OnEnable()`
      - `MonoBehaviour.Start()`
  - // other systems
- **FixedUpdate**
  - // other systems
  - RunNetworkFixedUpdate
    - `UpdateStage = NetworkUpdateStage.FixedUpdate`
    - `foreach (INetworkUpdateSystem in m_FixedUpdate_Array)`
      - `INetworkUpdateSystem.NetworkUpdate(UpdateStage)`
  - ScriptRunBehaviourFixedUpdate
    - `foreach (MonoBehaviour in m_Behaviours)`
      - `MonoBehaviour.FixedUpdate()`
  - // other systems
- **PreUpdate**
  - RunNetworkPreUpdate
    - `UpdateStage = NetworkUpdateStage.PreUpdate`
    - `foreach (INetworkUpdateSystem in m_PreUpdate_Array)`
      - `INetworkUpdateSystem.NetworkUpdate(UpdateStage)`
  - PhysicsUpdate
    - `GetPhysicsManager().Update();`
    - Rigidbody Interpolation
  - Physics2DUpdate
    - `GetPhysicsManager2D().Update();`
    - Rigidbody2D Interpolation
  - // other systems
- **Update**
  - RunNetworkUpdate
    - `UpdateStage = NetworkUpdateStage.Update`
    - `foreach (INetworkUpdateSystem in m_Update_Array)`
      - `INetworkUpdateSystem.NetworkUpdate(UpdateStage)`
  - ScriptRunBehaviourUpdate
    - `foreach (MonoBehaviour in m_Behaviours)`
      - `MonoBehaviour.Update()`
  - // other systems
- **PreLateUpdate**
  - // other systems
  - RunNetworkPreLateUpdate
    - `UpdateStage = NetworkUpdateStage.PreLateUpdate`
    - `foreach (INetworkUpdateSystem in m_PreLateUpdate_Array)`
      - `INetworkUpdateSystem.NetworkUpdate(UpdateStage)`
  - ScriptRunBehaviourLateUpdate
    - `foreach (MonoBehaviour in m_Behaviours)`
      - `MonoBehaviour.LateUpdate()`
- **PostLateUpdate**
  - // other systems
  - PlayerSendFrameComplete
    - `GetDelayedCallManager().Update(DelayedCallManager::kEndOfFrame);`
    - continue coroutine: `yield WaitForEndOfFrame`
  - RunNetworkPostLateUpdate
    - `UpdateStage = NetworkUpdateStage.PostLateUpdate`
    - `foreach (INetworkUpdateSystem in m_PostLateUpdate_Array)`
      - `INetworkUpdateSystem.NetworkUpdate(UpdateStage)`
  - // other systems

As seen above, player loop executes `Initialization` stage and that invokes `NetworkUpdateLoop`'s `RunNetworkInitialization` method which iterates over registered `INetworkUpdateSystem`s in `m_Initialization_Array` and calls `INetworkUpdateSystem.NetworkUpdate(UpdateStage)` on them.

In all stages, iterate over a static array and call `NetworkUpdate` method over `INetworkUpdateSystem` interface pattern is repeated.

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
    public void NetworkUpdate(NetworkUpdateStage updateStage)
    {
        Debug.Log($"{nameof(MyPlainScript)}.{nameof(NetworkUpdate)}({updateStage})");
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
    public void NetworkUpdate(NetworkUpdateStage updateStage)
    {
        Debug.Log($"{nameof(MyGameScript)}.{nameof(NetworkUpdate)}({updateStage})");
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

![](https://www.plantuml.com/plantuml/svg/tLf1L-Cs4BxpAto4m_AwlBsKRVjUI09DItPf0ilUzWbx4kj5bbn91jnVtv7i97OIqDZqKZWPIJCQZMQ-cT74hBZCcMPPDBUbWCXOpMDRPEB8R6Oo9LMSQfL1P7K1ZPa45gmGzb99E-V_GFJnqz6HL1OYzGWwkX6i17sjL7uUtKbko-Zifuy_dtwAhc_dZMLVm3uflRhB4sQXMZqhfEKPJ928Cu6SlyfhnP8fs5GbHti4qOVV3d7PaXycQSft1NPOQ0tIRaSFKFtouKFqweA09Cno9Kydqe2sn_N2zkG9cUcOQL5M0piS6pDqTfVy6PA3x1epT7Ot6WuosupJMHtaqqEUQ_pd9PcsSnrOAcDVvjcQF0bRu1mwcTvIgoBVCQoZ2F-sQtPtgBuhbhVeNe7b8wuXjahFMPOB2i70b3A-omLwgbiz5pnUsqs0Sew4230INNKYyy8Q-lgv3RF_Fcjun86rXn7ee7jwwcw6Yn8k8Vhrq5q9HvlzQC_y66ZDIZ__ucQJrkYyD1Qoy_RILnWuKph4mPE0J7PLvDRlSJbo2oihV5sxBHDKGbJ_3vMKQ5u4gzLgmGLeLAEsbnXrGdjSvOKpFsFHRaCMAvaeBO_DK6blIOPq2X_Cxk5X1dyNk0-9PlmCSM5XohYyJJMmv0_rOC97Zy6OgIeC-iKzbubw2f0C1J0zaHPq1Wb7iMPuJQ116mhXDPSG5WeCXiaBMAWrb0SePaXGtPnNyM21qjLSGehsx3skp51b3onUJPb4Tz8amhb5HMTSBhiGtSApBNG_MZoLGp-a2ggakfKlol95EP6FtF070IG0TpSbYtp8uXII4REC99epl2kyWcySdcReU2IY8PEFpBZbjvf9WpUU1Q6Mva5-aSTwcdvhLvX1fcgwhZY-2rBzG7oNIutWZU7_2tm78coiUMvHUoDmeub7toBVug_FtpLVfZPIYnRf5Ck52qZP8rU5FXqyiHKSdtIY-Ih7ag4xT2JpCjOq8TFxaykm4-3PELOe4sLmr3hh7pyYQ8KGyfR9fIAvmOYRdDauzMp_Ag00qWOtQMamQ7bSzLYVAvGFaesJOlB9Dw7rPVLXqLEpgftm-EGqpFJuv7F8U7BD4RQjO6l6jEG0jDk7oHWsWRactEOQZMBalufeN-398mgTzJerzXJq_BX8k7fzarSHR9Uq7uBjSgaHh37ypX1i0n2x9wVYIeCkY2ujoo3PqZKiMLpwzGF9Cry2a0Y_p-YRr6vEry0XQ6ZfuNCF1dEU5Q917PJtRFG6YSz6RvRPi-WizL0ec_O4apgFL6OjISSuxPduA3rhu-RDAydwgwZJ9ko7EMqUDU9i1jy7tjJtrTkxg4OwuqkUFAYaZBPHcb15hntg3RxKMSApKISPXod4Aoo_BZc8vAKSlqamcLxTJUPuVpvrIaYvqphOqzwbaBUllaci1nCIKrV7ToYIll44LCEH-tdncrrutt_VTxBG8qyWfWYxa3qg9Sjul0NjRkItpAmCCSJiPTEjbs-YURlWL0dHQTTDSdC1Iov9IL6tVEY5xSRx5l82uGWMeBN34IOAmekUJAuqorD4IVUMvv-rAhtGoXCbZTMCIzkfHi8IbyUYTW6lolgyGlI8x3kNtpkwx_3PHlZjn7i5YfsZKy5YIuQ0qN38Ljk8z1xeWjJwTLLZmKwqTl8kRyIhEf6OhxYKaFZSk66sf13OvG6osUIgdY-3zN_jGAuw3wpyTlczXvUEr_7-i8_IC8LyNVNH2sDLLZqpro_QH3K8GMqPZALmTWwoRjTt40ztgGrY2YO5OYFa7XvyHeQVY3p1qaZX9SZbbVChqpNXwgIpAaMZkLsbt4c61o1PyZZZjhBZAr0PM5rszxTBBhb9IIgebbQ-xIKDzp8kt0lcL2MzC6ketECPfI2fqMV8XDWhclRdpizNw4jV5twJOCKIApHWMG1-eBNW9vn7fQj2edl6xQNIMrwTW-m-eFUCilkCrc6sDVC-ukmQm7FVYuirW3IfihJUf8VDaMQqPcdMiLEwOy7-hQP3DtC6Hec2K0xxCdLSxn9gEQ1PezLlpTIvWckNXWw31RJN0gd0dQSCDVGxauc0iTOggAgwcO3d63ppODl4jyKXkQuwg2DuA9RKtmQqVmyIscaAcTAbrvDhYs84Spq7MWFVzL27u5urg2fU2xcvIhISVm40)

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
NetworkUpdateLoop -> NetworkUpdateLoop: Initialization.Insert(NetworkInitialization)
NetworkUpdateLoop -> NetworkUpdateLoop: EarlyUpdate.Insert(NetworkEarlyUpdate)
NetworkUpdateLoop -> NetworkUpdateLoop: FixedUpdate.Insert(NetworkFixedUpdate)
NetworkUpdateLoop -> NetworkUpdateLoop: PreUpdate.Insert(NetworkPreUpdate)
NetworkUpdateLoop -> NetworkUpdateLoop: Update.Insert(NetworkUpdate)
NetworkUpdateLoop -> NetworkUpdateLoop: PreLateUpdate.Insert(NetworkPreLateUpdate)
NetworkUpdateLoop -> NetworkUpdateLoop: PostLateUpdate.Insert(NetworkPostLateUpdate)
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
    PlayerLoop -> NetworkUpdateLoop: RunNetworkEarlyUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = EarlyUpdate
    loop m_EarlyUpdate_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
    PlayerLoop -> PlayerLoop: ScriptRunDelayedStartupFrame
    note right of PlayerLoop: MonoBehaviour.Awake()
    note right of PlayerLoop: MonoBehaviour.OnEnable()
    note right of PlayerLoop: MonoBehaviour.Start()
    PlayerLoop -> PlayerLoop: UpdateKinect
    PlayerLoop -> PlayerLoop: DeliverIosPlatformEvents
    PlayerLoop -> PlayerLoop: TangoUpdate
    PlayerLoop -> PlayerLoop: DispatchEventQueueEvents
    PlayerLoop -> PlayerLoop: PhysicsResetInterpolatedTransformPosition
    note right of PlayerLoop: GetPhysicsManager().ResetInterpolatedTransformPosition();
    PlayerLoop -> PlayerLoop: SpriteAtlasManagerUpdate
    PlayerLoop -> PlayerLoop: PerformanceAnalyticsUpdate
end
group FixedUpdate
    PlayerLoop -> PlayerLoop: ClearLines
    PlayerLoop -> PlayerLoop: NewInputFixedUpdate
    PlayerLoop -> PlayerLoop: DirectorFixedSampleTime
    PlayerLoop -> PlayerLoop: AudioFixedUpdate
    PlayerLoop -> NetworkUpdateLoop: RunNetworkFixedUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = FixedUpdate
    loop m_FixedUpdate_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
    PlayerLoop -> PlayerLoop: ScriptRunBehaviourFixedUpdate
    note right of PlayerLoop: MonoBehaviour.FixedUpdate()
    PlayerLoop -> PlayerLoop: DirectorFixedUpdate
    PlayerLoop -> PlayerLoop: LegacyFixedAnimationUpdate
    PlayerLoop -> PlayerLoop: XRFixedUpdate
    PlayerLoop -> PlayerLoop: PhysicsFixedUpdate
    note right of PlayerLoop: GetPhysicsManager().FixedUpdate();
    note right of PlayerLoop: GetPhysicsManager().Simulate();
    PlayerLoop -> PlayerLoop: Physics2DFixedUpdate
    note right of PlayerLoop: GetPhysicsManager2D().FixedUpdate();
    note right of PlayerLoop: GetPhysicsManager2D().Simulate();
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
    note right of PlayerLoop: GetPhysicsManager().Update();
    note right of PlayerLoop: Rigidbody Interpolation
    PlayerLoop -> PlayerLoop: Physics2DUpdate
    note right of PlayerLoop: GetPhysicsManager2D().Update();
    note right of PlayerLoop: Rigidbody2D Interpolation
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
    note right of PlayerLoop: MonoBehaviour.Update()
    PlayerLoop -> PlayerLoop: ScriptRunDelayedDynamicFrameRate
    PlayerLoop -> PlayerLoop: ScriptRunDelayedTasks
    PlayerLoop -> PlayerLoop: DirectorUpdate
end
group PreLateUpdate
    PlayerLoop -> PlayerLoop: AIUpdatePostScript
    PlayerLoop -> PlayerLoop: DirectorUpdateAnimationBegin
    PlayerLoop -> PlayerLoop: LegacyAnimationUpdate
    PlayerLoop -> PlayerLoop: DirectorUpdateAnimationEnd
    PlayerLoop -> PlayerLoop: DirectorDeferredEvaluate
    PlayerLoop -> PlayerLoop: EndGraphicsJobsAfterScriptUpdate
    PlayerLoop -> PlayerLoop: ConstraintManagerUpdate
    PlayerLoop -> PlayerLoop: ParticleSystemBeginUpdateAll
    PlayerLoop -> NetworkUpdateLoop: RunNetworkPreLateUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = PreLateUpdate
    loop m_PreLateUpdate_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
    PlayerLoop -> PlayerLoop: ScriptRunBehaviourLateUpdate
    note right of PlayerLoop: MonoBehaviour.LateUpdate()
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
    note right of PlayerLoop: GetDelayedCallManager().Update(DelayedCallManager::kEndOfFrame);
    note right of PlayerLoop: continue coroutine: yield WaitForEndOfFrame
    PlayerLoop -> NetworkUpdateLoop: RunNetworkPostLateUpdate
    NetworkUpdateLoop -> NetworkUpdateLoop: UpdateStage = PostLateUpdate
    loop m_PostLateUpdate_Array
        NetworkUpdateLoop -> INetworkUpdateSystem: NetworkUpdate
        NetworkUpdateLoop <-- INetworkUpdateSystem
    end
    PlayerLoop <-- NetworkUpdateLoop
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
end
```
</details>

### Pseudo-code

```cs
public interface INetworkUpdateSystem
{
    void NetworkUpdate(NetworkUpdateStage updateStage);
}

public enum NetworkUpdateStage : byte
{
    Initialization = 1,
    EarlyUpdate = 2,
    FixedUpdate = 3,
    PreUpdate = 4,
    Update = 0,
    PreLateUpdate = 5,
    PostLateUpdate = 6
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
                {
                    // insert at the bottom of `Initialization`
                    subsystems.Add(NetworkInitialization.CreateLoopSystem());
                }
                playerLoopSystem.subSystemList = subsystems.ToArray();
            }
            else if (playerLoopSystem.type == typeof(EarlyUpdate))
            {
                var subsystems = playerLoopSystem.subSystemList.ToList();
                {
                    int indexOfScriptRunDelayedStartupFrame = subsystems.FindIndex(s => s.type == typeof(EarlyUpdate.ScriptRunDelayedStartupFrame));
                    if (indexOfScriptRunDelayedStartupFrame < 0)
                    {
                        Debug.LogError($"{nameof(NetworkUpdateLoop)}.{nameof(Initialize)}: Cannot find index of '{nameof(EarlyUpdate.ScriptRunDelayedStartupFrame)}' under '{nameof(EarlyUpdate)}' subsystems!");
                        return;
                    }

                    // insert before `EarlyUpdate.ScriptRunDelayedStartupFrame`
                    subsystems.Insert(indexOfScriptRunDelayedStartupFrame, NetworkEarlyUpdate.CreateLoopSystem());
                }
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
            m_Initialization_Array[i].NetworkUpdate(UpdateStage);
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
            m_EarlyUpdate_Array[i].NetworkUpdate(UpdateStage);
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

## Network Time, Tick & Frames

Relying on this pipeline, we could implement network time and tick numbers to reference specific network frames — potentially for interpolation, snapshotting, rollback/replay etc.

## Transport IO

A transport system could be polled very early in the `EarlyUpdate` stage to receive input data and could be flushed on `PostLateUpdate` stage to send output data.

## RPC Execution

RPCs might carry `NetworkUpdateStage` information with them and be executed at specific stages on the remote-end.

## Network Variables & Snapshotting

Network variables need to know when they should be written and when they should be read, so that they could snapshot themselves for outbound and update themselves for inbound.

## Other Tick/Update Dependent Network Systems

With this network update loop pipeline, it's natural to register any tick/frame/update based network systems into relevant stages. We are expecting lots of upcoming network systems utilizing this infrastructure in the future.
