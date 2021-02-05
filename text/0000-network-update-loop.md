- Feature Name: `network-update-loop`
- Start Date: 2021-01-31
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

Often there is a need to update netcode systems like RPC queue, transport IO and others outside standard [`MonoBehaviour` event cycle](https://docs.unity3d.com/Manual/ExecutionOrder.html).

This RFC proposes a new **Network Update Loop** infrastructure that utilizes [Unity's low-level Player Loop API](https://docs.unity3d.com/ScriptReference/LowLevel.PlayerLoop.html) and allows for registering `INetworkUpdateSystem`s with `NetworkUpdate()` methods to be executed at specific `NetworkUpdateStage`s which may also be prior to or after `MonoBehaviour`-driven game logic execution.

Implementation is expected to have minimal yet flexible API that would allow further systems such as network tick to be easily developed.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the Unity Multiplayer and you were teaching it to another Unity developer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Unity developers should _think_ about the feature, and how it should impact the way they develop multiplayer projects in Unity. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Unity developers and new Unity developers.

For implementation-oriented RFCs (e.g. for framework internals), this section should focus on how Unity Multiplayer contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

## Injecting NetworkUpdateLoop Systems Into PlayerLoop

![](https://www.plantuml.com/plantuml/svg/tLbBSzis4BxxL_2OJ1fFaicbcROpiYIhQgN45UKajmnChaICGO05GDlih-y2f2JgYQLyYdsIYVqMu3w-NIpDFPS5qooFObi9Y8pLFB5bBYxCUYLNKYMPLZb4LfVGMabXmKfXhvMqCVyFelSVBowaiX2f1z3HZM0Lw8bInQVeLd9RHH_UlV_rycCqcSSjiSzW7vM-lorIPg6MdfNIyXp62E9CeEOlSg9OEWMRgVHY3n2z_UEGbWtyCAHzlX7OVgarIBjvUW3rszktqKc1m4cOvKfUzjBqLIErlVAQh778jAZg1vTZQK3RL-z-MGcs5kaQh_aJfCSqjXRf6cyq7DEtNg9hEwRgZ5DDy9yJ9ziiTz2gZ6sfRNbuU-LrE1HZw1I3IwtAOYV2BYBysyZixb3_hbNECSy2uWEk8R5Anoml10NGCvLCRWk1AUkaOy6JLVG4a342aMaavknO5gKrTFdOXjL_dxLIuu3iOIewcAeUJOsENlh0P0BNo9dG2VZcZSVq5H9Y9s3z6ssoXEZtGz4X73R_iRtoewSrg_pwqSt6Mq_ZobZ8FssBdb3WWlUG1Wy1cUuwnSdlQRbmWdNSFyEVkz3pJBkhfR5hw_7FOjG0PoWXwl_xIaha0mBDQBNW0ZIQIjjDx1y7Z8hIyOMnI3GXYf5CL9Odfg-qJtr3uXVVXdo3kndyM-8-9Cpm2PBImfHnESyqiFGlTMV2hvw0CLDL6YIoRrocwZ40wGjWUYmjw1nIZiUQeUT0WpOKu8HB21yA33OuLx1Gg_8RGCmGK7lEcxYtmDCrMaA6zkfz8PYXoUvOSZzb4bD6au2nHKFdN2wxWERmP4jz7XRVofKhH0LDKdlbix9yKOtb0pVyJW2fuZfFoRBeioI3TC0i8ntD1Tu8NV2Lm-CC4uy4wZFvm8mJdsic3Ney3zP6MWPv8L-SQVPjKc46kMfmePgUQ5vOvEa1kAUffu-oe118VRY_kGoR2PLpl7sCbS6drjbd-81X7SgXEPDBHJBKW9k2sIJpQlyfeGJI1ZTPPP3MVEPvDYWKdW_IjHknd-G4yeqBpJ41hM2cyHFfr3UWtIQkMrbJR0FS5DtMRQ1hqTs5dbTqTxKqHRVr-6KKNJArVuP7drgdFNEegVS8UcTVfbpHDrcdyXLax86hKfzvG19Y0akML5w-9tdkEreIyFkClacJdvq17mQT8NsXRFOiWxcxPwEi8-bw9PvoPzkvVtEYTJYP-Tl0y_9-MqUJzUk7BwJHhltLBRP8qlqCadjiaguvY9HIYSVJqUqunlVovIibrQyL4z7u0NNx7fcakZScANNTTeTMvYjFGRK2_BpeVdPexyVrImhg_KBGhNuqh3Ie97RsoOQNki7dpDmR8elkDJrRWy4po6y7V3O1xkZPaE1cuMN4p2hTNH6hFTd9TUiMu0gMN99QXsvTmv7jXljlVmGuW0LexNYLYP8mXv3fIBCYGy9t4mX_6Q_jGcD_9OtLU9UqdUu3UFVYYMZSwqtPlAyGdNE-5RgjlDgQZvFkqQvihAOI6kz6H7kJTtacxJHLERxdKaBgkxYMuKX5gSl5lJtDhw_zmUj2uqBNp0Salrv_FoDEXgiA78wIur5DTbVpwgMnAg_7CyqaY6W68QYZjg4KV9bPaCq7oWx7mrsQW9Y2oO0uGFm6Txv7mf-Oii4cNg4Jv7BayifqRO4wsQnkccXfLnjtdECBa29vc7DRn_48L0vMLztpxPfBRZAI2MhHEhVzISCpm2itrvcgz2pEwYjtrwO6DIlyI3FH0D9aoj96l3wB1fhHPivvnu7ppWlvIsQ7eV0CZ18bOOJbxn73PfVGN4xp9TOqbFwogCtCX71Jvz3Bu0t7Mu5q0uapXXRwaybKm51h5JHLCyR3z-WzyU1iqNzTW7HI3w87S5SkgPzOjh-3XdQQGgvqvUtafij8Q6xWH_iEuaRHVN2JIQg9LuSyuShJpVD-Xt1czxolQDK-RwtyH9z1_mS0)

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
    NetworkUpdateLoop -> NetworkUpdateLoop: AdvanceFrame
    NetworkUpdateLoop -> NetworkUpdateLoop: ++FrameCount
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

    public static uint FrameCount = 0;
    public static NetworkUpdateStage UpdateStage;

    private static void AdvanceFrame()
    {
        ++FrameCount;
    }

    private static readonly List<INetworkUpdateSystem> m_Initialization_List = new List<INetworkUpdateSystem>();
    private static INetworkUpdateSystem[] m_Initialization_Array = new INetworkUpdateSystem[0];

    private static void RunNetworkInitialization()
    {
        AdvanceFrame();

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
    public static uint FrameCount;
    public static NetworkUpdateStage UpdateStage;

    public static void RegisterAllNetworkUpdates(this INetworkUpdateSystem updateSystem);
    public static void RegisterNetworkUpdate(this INetworkUpdateSystem updateSystem, NetworkUpdateStage updateStage);
    public static void UnregisterNetworkUpdate(this INetworkUpdateSystem updateSystem, NetworkUpdateStage updateStage);
    public static void UnregisterAllNetworkUpdates(this INetworkUpdateSystem updateSystem);
}

public class MyPlainScript : IDisposable, INetworkUpdateSystem
{
    public void Initialize()
    {
        this.RegisterNetworkUpdate(NetworkUpdateStage.EarlyUpdate);
        this.RegisterNetworkUpdate(NetworkUpdateStage.FixedUpdate);
        this.RegisterNetworkUpdate(NetworkUpdateStage.Update);
    }

    public void NetworkUpdate()
    {
        Debug.Log($"{nameof(MyPlainScript)}.{nameof(NetworkUpdate)}()");
    }

    public void Dispose()
    {
        this.UnregisterAllNetworkUpdates();
    }
}

public class MyGameScript : MonoBehaviour, INetworkUpdateSystem
{
    private void OnEnable()
    {
        this.RegisterNetworkUpdate(NetworkUpdateStage.EarlyUpdate);
        this.RegisterNetworkUpdate(NetworkUpdateStage.FixedUpdate);
        this.RegisterNetworkUpdate(NetworkUpdateStage.Update);
    }

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
        this.UnregisterAllNetworkUpdates();
    }
}
```

# Drawbacks
[drawbacks]: #drawbacks

Why should we _not_ do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

- For framework, tools, and library proposals: Does this feature exist in other networking stacks and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other projects, provide readers of your RFC with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other projects.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC. Please also take into consideration that Unity Multiplayer sometimes intentionally diverges from common multiplayer networking features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the Unity Multiplayer as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the Unity Multiplayer in your proposal. Also consider how the this all fits into the roadmap for the project and the team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.
