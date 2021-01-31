- Feature Name: `network-ticker`
- Start Date: 2021-01-31
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

One paragraph explanation of the feature.

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

## NetworkTicker Injecting Into PlayerLoop

![](https://www.plantuml.com/plantuml/svg/rLbVazes47-_J-77PD9aaanUccaxmm57QO_aYhaaRncTlM0DikHAyjsvdxuhsO0n1rgJbnmlWBM_rMh_QstMN1PCipoARIKWCbRpn9Qvk33RbLn9bMHQvH1PNK9h9OKbAuMzKjB6_3-8tdsuk93AGgJkGKStWbMWhvKgFaQtajigUl_stQzlFaJNYxX5zWdiWzBh1K_Me3z6yr9QdiCK0Pm9vFfPNYkMpi1cAhtO8uvU_z55xs27f6rv9c27fTOWxUvf1_7RwzVHqSKJEpFBbNmpIDCN3SN-oiiWfp7JehejKcRXpLGQqFR5srysHAt5CASh_WZfCKwjnJ2d6mt7-6oNmboEiUWXnJKIdf1ajqpWfenjiMjPR7_bvMbNYe7iGokjogBZAHSHVdOqNUxrxrgG1SP5W7p4DIHMwPZbXQ001gQIoMqXCCGSTInuil6A039686O5YR4sbKLf3Pq-jc7st-OjLt-2jcDCz_ral7beo1K6wJsJ2RY_9FfNhzoleS9SHQ9tttZe5KI_FsL-uSwKVzzV-Uv0x4x0G6jMURedjvgsdZZoZssI8p3dbrsS9mMPTdNjQFVG5uweTkLpJTJ5zT5V5zieh_NuvzLS8DQL4FN_eP8IuvYG7hHQSW4QbQFj5TRaK5XDYjB5kR6O84AfGJ9HMPwOeT8-P0qfB_XI_nNkvl1lYVkGCED7I4eBCyRb8jF0qh_KdGaVFG5ZPgeqc82-SfcgXm1aA83fgRIWSqWvYZL7gu46RIZ0QouXB1GOR2ekOQasPJ60c2EWjfnNyM29qZLSGehsx7q2dwFAxbYodZ69GQ99mDYYedCkLtr02tYqfJuFYwToPGPHGPNKLlcaB5zMKtdF3T-4MQXIEal9iXYo98Dqn2mZYUOSRu8c-CBXOOw1Hm8DcRndnf4lD9C6BPu7QnejWZoHBweq_JQXC8TSDRXGBKuqBoossYDmJrEVFig2aa1guVxcCgmcPCwnmvygWq-jqy_u7jqw54CB9bUAfAWHDmMpIURP_bD22QGDRhBAO5hpaUUpGQ5GFaYtPiJ-ZnEIRrneZW5e1JEDdKXItu1scxY6OvjY6-2kihQwyrwLjOlhKrcxsAQojXu_ryfApj-VuC77psdrFzLnUmIzxIvJhkYRREFr4Z9sprMfpnIGXBY65KigJp-GFFVTA0duRKxVf0d2te3tezwGeL0s-o65kGjXmwndPlLCV5wTxmVupOd-u7XuxwFw1FS-iWdjxSDdspBNHkddG2JgOGR96jkTAmuYfTJKwMnoEutnJBuGfQHAj4atj2I0l7stIs8zc9AOkhwvHrFoXQUWMbvyXWkVwR-7plkyFVSmfVUh14r-6LKI6_5abs7wmHVCh0qn7nxqA7lpjhvfh8FicSeww5qwsoqyLyVO1AwhERMvUjbdMx4lOSKbgTBtA_97jXiVdlm8S0HBqDfbI2PAmcn0fXFDYWnJjviA-0johHTQ7bVIM8ttEzkhHyVx44y4n9Mna7jLDkSLedTWjvor5JnxCsRItTOLWCsaGEDzXQXlyYu_c5kQ-XglkPIG-gQhfJXIUUbpcMvFnkkhUB2xRqpeUky0vPUhRsVuoNXJIyDUSjohoUPgZbuQg_9wPD9CPuXg488w_XfBmLUP1TcydEiX7kxIECGCJ0R4eTvNkFFT3lx4a0tst0gD85SJ5rUcRpFJIsTr-qCBkrgvqnmFGFBaSSvjxSSJK3bONVMFjoikkSb8AgX9wzZsaeRdSfTkrZ9JwLavQwZSrPZgrApmOyP40qYJAKiRk_ee6cb6cmFdFOV0dGFvEs6FGUqSZ18bOQpaRmJZPfVGh4pp5TOqfFwyhDNCX72JuD1XyAxYjK2wIyGPGmtzoMGgOAor2ggg6QtXEGQFF3ZlynD_IA_huV08xieLzRN7_Zl5q4upo9Mkl9xSn1SJcXl2qTvDkF7m4DoqaAhYZP0dJ7cYGpvi3tgro8VmJO-yj_9JlsJy7m00)

<details>
  <summary>UML Source</summary>

```
skinparam Style strictuml
skinparam monochrome true
skinparam defaultFontSize 14

note over PlayerLoop: Unity 2019.4 LTS
note over NetworkTicker: RuntimeInitializeOnLoadMethod
NetworkTicker -> NetworkTicker: Initialize
NetworkTicker -> PlayerLoop: GetCurrentPlayerLoop
NetworkTicker <-- PlayerLoop
NetworkTicker -> NetworkTicker: Initialization.Add(NetworkInitialization)
NetworkTicker -> NetworkTicker: EarlyUpdate.Insert(0, NetworkEarlyUpdate)
NetworkTicker -> NetworkTicker: FixedUpdate.Insert(0, NetworkFixedUpdate)
NetworkTicker -> NetworkTicker: PreUpdate.Insert(0, NetworkPreUpdate)
NetworkTicker -> NetworkTicker: Update.Insert(0, NetworkUpdate)
NetworkTicker -> NetworkTicker: PreLateUpdate.Insert(0, NetworkPreLateUpdate)
NetworkTicker -> NetworkTicker: PostLateUpdate.Add(NetworkPostLateUpdate)
NetworkTicker -> PlayerLoop: SetPlayerLoop
NetworkTicker <-- PlayerLoop
group Initialization
    PlayerLoop -> PlayerLoop: PlayerUpdateTime
    PlayerLoop -> PlayerLoop: DirectorSampleTime
    PlayerLoop -> PlayerLoop: AsyncUploadTimeSlicedUpdate
    PlayerLoop -> PlayerLoop: SynchronizeInputs
    PlayerLoop -> PlayerLoop: SynchronizeState
    PlayerLoop -> PlayerLoop: XREarlyUpdate
    PlayerLoop -> NetworkTicker: TickNetworkInitialization
    NetworkTicker -> NetworkTicker: AdvanceTick
    NetworkTicker -> NetworkTicker: ++TickCount
    NetworkTicker -> NetworkTicker: TickStage = Initialization
    loop m_Initialization_TickableArray
        NetworkTicker -> INetworkTickable: NetworkTick
        NetworkTicker <-- INetworkTickable
    end
    PlayerLoop <-- NetworkTicker
end
group EarlyUpdate
    PlayerLoop -> NetworkTicker: TickNetworkEarlyUpdate
    NetworkTicker -> NetworkTicker: TickStage = EarlyUpdate
    loop m_EarlyUpdate_TickableArray
        NetworkTicker -> INetworkTickable: NetworkTick
        NetworkTicker <-- INetworkTickable
    end
    PlayerLoop <-- NetworkTicker
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
    PlayerLoop -> NetworkTicker: TickNetworkFixedUpdate
    NetworkTicker -> NetworkTicker: TickStage = FixedUpdate
    loop m_FixedUpdate_TickableArray
        NetworkTicker -> INetworkTickable: NetworkTick
        NetworkTicker <-- INetworkTickable
    end
    PlayerLoop <-- NetworkTicker
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
    PlayerLoop -> NetworkTicker: TickNetworkPreUpdate
    NetworkTicker -> NetworkTicker: TickStage = PreUpdate
    loop m_PreUpdate_TickableArray
        NetworkTicker -> INetworkTickable: NetworkTick
        NetworkTicker <-- INetworkTickable
    end
    PlayerLoop <-- NetworkTicker
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
    PlayerLoop -> NetworkTicker: TickNetworkUpdate
    NetworkTicker -> NetworkTicker: TickStage = Update
    loop m_Update_TickableArray
        NetworkTicker -> INetworkTickable: NetworkTick
        NetworkTicker <-- INetworkTickable
    end
    PlayerLoop <-- NetworkTicker
    PlayerLoop -> PlayerLoop: ScriptRunBehaviourUpdate
    PlayerLoop -> PlayerLoop: ScriptRunDelayedDynamicFrameRate
    PlayerLoop -> PlayerLoop: ScriptRunDelayedTasks
    PlayerLoop -> PlayerLoop: DirectorUpdate
end
group PreLateUpdate
    PlayerLoop -> NetworkTicker: TickNetworkPreLateUpdate
    NetworkTicker -> NetworkTicker: TickStage = PreLateUpdate
    loop m_PreLateUpdate_TickableArray
        NetworkTicker -> INetworkTickable: NetworkTick
        NetworkTicker <-- INetworkTickable
    end
    PlayerLoop <-- NetworkTicker
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
    PlayerLoop -> NetworkTicker: TickNetworkPostLateUpdate
    NetworkTicker -> NetworkTicker: TickStage = PostLateUpdate
    loop m_PostLateUpdate_TickableArray
        NetworkTicker -> INetworkTickable: NetworkTick
        NetworkTicker <-- INetworkTickable
    end
    PlayerLoop <-- NetworkTicker
end
```
</details>

### Pseudo-code

```cs
public interface INetworkTickable
{
    void NetworkTick();
}

public enum NetworkTickStage
{
    Initialization = -4,
    EarlyUpdate = -3,
    FixedUpdate = -2,
    PreUpdate = -1,
    Update = 0,
    PreLateUpdate = 1,
    PostLateUpdate = 2
}

public static class NetworkTicker
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
                updateDelegate = TickNetworkInitialization
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
                updateDelegate = TickNetworkEarlyUpdate
            };
        }
    }

    // define other loop systems ...

    public static uint TickCount = 0;
    public static NetworkTickStage TickStage;

    private static void AdvanceTick()
    {
        ++TickCount;
    }

    private static readonly List<INetworkTickable> m_Initialization_TickableList = new List<INetworkTickable>();
    private static INetworkTickable[] m_Initialization_TickableArray = new INetworkTickable[0];

    private static void TickNetworkInitialization()
    {
        AdvanceTick();

        TickStage = NetworkTickStage.Initialization;
        int tickableArrayLength = m_Initialization_TickableArray.Length;
        for (int i = 0; i < tickableArrayLength; i++)
        {
            m_Initialization_TickableArray[i].NetworkTick();
        }
    }

    private static readonly List<INetworkTickable> m_EarlyUpdate_TickableList = new List<INetworkTickable>();
    private static INetworkTickable[] m_EarlyUpdate_TickableArray = new INetworkTickable[0];

    private static void TickNetworkEarlyUpdate()
    {
        TickStage = NetworkTickStage.EarlyUpdate;
        int tickableArrayLength = m_EarlyUpdate_TickableArray.Length;
        for (int i = 0; i < tickableArrayLength; i++)
        {
            m_EarlyUpdate_TickableArray[i].NetworkTick();
        }
    }

    // implement other loop systems ...

    public static void RegisterNetworkTick(this INetworkTickable tickable, NetworkTickStage tickStage = NetworkTickStage.Update)
    {
        switch (tickStage)
        {
            case NetworkTickStage.Initialization:
            {
                if (!m_Initialization_TickableList.Contains(tickable))
                {
                    m_Initialization_TickableList.Add(tickable);
                    m_Initialization_TickableArray = m_Initialization_TickableList.ToArray();
                }

                break;
            }
            case NetworkTickStage.EarlyUpdate:
            {
                if (!m_EarlyUpdate_TickableList.Contains(tickable))
                {
                    m_EarlyUpdate_TickableList.Add(tickable);
                    m_EarlyUpdate_TickableArray = m_EarlyUpdate_TickableList.ToArray();
                }

                break;
            }
            // register other loop systems ...
        }
    }

    public static void UnregisterNetworkTick(this INetworkTickable tickable, NetworkTickStage tickStage = NetworkTickStage.Update)
    {
        switch (tickStage)
        {
            case NetworkTickStage.Initialization:
            {
                if (m_Initialization_TickableList.Contains(tickable))
                {
                    m_Initialization_TickableList.Remove(tickable);
                    m_Initialization_TickableArray = m_Initialization_TickableList.ToArray();
                }

                break;
            }
            case NetworkTickStage.EarlyUpdate:
            {
                if (m_EarlyUpdate_TickableList.Contains(tickable))
                {
                    m_EarlyUpdate_TickableList.Remove(tickable);
                    m_EarlyUpdate_TickableArray = m_EarlyUpdate_TickableList.ToArray();
                }

                break;
            }
            // unregister other loop systems ...
        }
    }
}
```

## NetworkTicker Ticking INetworkTickables

![](https://www.plantuml.com/plantuml/svg/xLPTJy8m57tVh-YZFZ1YuXDH4unWI4GDasTIjoiqTEtITUZyzMxZOstNCRP4Ouoy0BtddjETUzlTU4rOX0KEaITJ2YYMWlWo2QaJ7o8XPznV2Hu2aY819HB06qwe77CcFV89wEBISHYNWFW619gcpnGJvlc2H7A09dSaZdYCNoaS0Js2VETY_KByTGLvZqFO0wVPfcvXXJU49w8MLQ5R2fv4kgWxOKGIJBC7S53sqOAeTuCK3X03D8CbYIK8PVbiX0LDvr609PnRIAvwFPsbiz2OV43m4q9jD805UsFLghXFRCGArxSaPM6wkwfmr3rhQncBfzyXqqAXD3GpFWNnm7daAcuK76N8ie7yUxTavcd8cbHFuYMWQsJcqbmjN2ZBY_tH6Wg1qm9a5J7EkHAloSbTqPAESQjd_bJgCgU0vPuRjjfBh_jU_XkWVX-vhckldj9ahQfdvhMfdfcxgvwo_9UhPo_EST2MSPQmmoLcUcYxvqnCLKD_HXlUl53q36NpbXxjxeQrLJjqQSS6hVRc_wNIt98DtTYY4NzP3vhJGpOmdZe-p9dOlNA7b2gnkDthLfbHtUtFqsR29ld6-UaB)

<details>
  <summary>UML Source</summary>

```
skinparam Style strictuml
skinparam monochrome true
skinparam defaultFontSize 14

note over MyPlainScript: IDisposable
note over MyPlainScript: INetworkTickable
note over MyGameScript: MonoBehaviour
note over MyGameScript: INetworkTickable
group MyPlainScript.Initialize
    MyPlainScript -> NetworkTicker: RegisterNetworkTick(EarlyUpdate)
    MyPlainScript <-- NetworkTicker
    MyPlainScript -> NetworkTicker: RegisterNetworkTick(FixedUpdate)
    MyPlainScript <-- NetworkTicker
    MyPlainScript -> NetworkTicker: RegisterNetworkTick(Update)
    MyPlainScript <-- NetworkTicker
end
group MonoBehaviour.OnEnable
    MyGameScript -> NetworkTicker: RegisterNetworkTick(EarlyUpdate)
    MyGameScript <-- NetworkTicker
    MyGameScript -> NetworkTicker: RegisterNetworkTick(FixedUpdate)
    MyGameScript <-- NetworkTicker
    MyGameScript -> NetworkTicker: RegisterNetworkTick(Update)
    MyGameScript <-- NetworkTicker
end
group PlayerLoop.EarlyUpdate
    PlayerLoop -> NetworkTicker: TickNetworkEarlyUpdate
    NetworkTicker -> NetworkTicker: TickStage = EarlyUpdate
    loop m_EarlyUpdate_TickableArray
        NetworkTicker -> MyPlainScript: NetworkTick
        NetworkTicker <-- MyPlainScript
        NetworkTicker -> MyGameScript: NetworkTick
        NetworkTicker <-- MyGameScript
    end
    PlayerLoop <-- NetworkTicker
    PlayerLoop -> PlayerLoop: // ...
end
group PlayerLoop.FixedUpdate
    PlayerLoop -> NetworkTicker: TickNetworkFixedUpdate
    NetworkTicker -> NetworkTicker: TickStage = FixedUpdate
    loop m_FixedUpdate_TickableArray
        NetworkTicker -> MyPlainScript: NetworkTick
        NetworkTicker <-- MyPlainScript
        NetworkTicker -> MyGameScript: NetworkTick
        NetworkTicker <-- MyGameScript
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
    PlayerLoop -> NetworkTicker: TickNetworkUpdate
    NetworkTicker -> NetworkTicker: TickStage = Update
    loop m_Update_TickableArray
        NetworkTicker -> MyPlainScript: NetworkTick
        NetworkTicker <-- MyPlainScript
        NetworkTicker -> MyGameScript: NetworkTick
        NetworkTicker <-- MyGameScript
    end
    PlayerLoop <-- NetworkTicker
    PlayerLoop -> PlayerLoop: ScriptRunBehaviourUpdate
    group MonoBehaviour.Update
        PlayerLoop -> MyGameScript: Update
        MyGameScript -> MyGameScript: // ...
        PlayerLoop <-- MyGameScript
    end
    PlayerLoop -> PlayerLoop: // ...
end
group MonoBehaviour.OnDisable
    MyGameScript -> NetworkTicker: UnregisterAllNetworkTicks
    MyGameScript <-- NetworkTicker
end
group IDisposable.Dispose
    MyPlainScript -> NetworkTicker: UnregisterAllNetworkTicks
    MyPlainScript <-- NetworkTicker
end
```
</details>

### Pseudo-code

```cs
public interface INetworkTickable
{
    void NetworkTick();
}

public enum NetworkTickStage
{
    Initialization = -4,
    EarlyUpdate = -3,
    FixedUpdate = -2,
    PreUpdate = -1,
    Update = 0,
    PreLateUpdate = 1,
    PostLateUpdate = 2
}

public static class NetworkTicker
{
    public static uint TickCount;
    public static NetworkTickStage TickStage;

    public static void RegisterAllNetworkTicks(this INetworkTickable tickable) { /* ... */ }
    public static void RegisterNetworkTick(this INetworkTickable tickable, NetworkTickStage tickStage) { /* ... */ }
    public static void UnregisterNetworkTick(this INetworkTickable tickable, NetworkTickStage tickStage) { /* ... */ }
    public static void UnregisterAllNetworkTicks(this INetworkTickable tickable) { /* ... */ }
}

public class MyPlainScript : IDisposable, INetworkTickable
{
    public void Initialize()
    {
        this.RegisterNetworkTick(NetworkTickStage.EarlyUpdate);
        this.RegisterNetworkTick(NetworkTickStage.FixedUpdate);
        this.RegisterNetworkTick(NetworkTickStage.Update);
    }

    public void NetworkTick()
    {
        Debug.Log($"{nameof(MyPlainScript)}.{nameof(NetworkTick)}({NetworkTicker.TickStage})");
    }

    public void Dispose()
    {
        this.UnregisterAllNetworkTicks();
    }
}

public class MyGameScript : MonoBehaviour, INetworkTickable
{
    private void OnEnable()
    {
        this.RegisterNetworkTick(NetworkTickStage.EarlyUpdate);
        this.RegisterNetworkTick(NetworkTickStage.FixedUpdate);
        this.RegisterNetworkTick(NetworkTickStage.Update);
    }

    public void NetworkTick()
    {
        print($"{nameof(MyGameScript)}.{nameof(NetworkTick)}({NetworkTicker.TickStage})");
    }

    private void FixedUpdate()
    {
        print($"{nameof(MyGameScript)}.{nameof(FixedUpdate)}()");
    }

    private void Update()
    {
        print($"{nameof(MyGameScript)}.{nameof(Update)}()");
    }

    private void OnDisable()
    {
        this.UnregisterNetworkTick();
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
