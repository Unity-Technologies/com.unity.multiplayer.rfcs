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

![](https://www.plantuml.com/plantuml/svg/rLbVSzeu47_Ff_1ZExrssavVhdDtCmG8nnrfSvYqVUieze8ro99FafFuFlsjP0E6X6X5NnfU06j_gzN-rzgikQoOPcMIsaP0OgpccQrBSMQsAwMIAYkqAY6nkeRUIWvBLWjxeQHD-N-GlFjmTYQLXKJTWawk16j0pvIgFYRNajicUV_stQzlFYIpHTeZ-mJsJkdrWcThq1-JAL9o9f3TPpbJB3_RMqzU_x77Tm9sM6iDqkwU7Y1-U_qwERg8x8xjponNyiqWprzq5FihBuESnaoBvhhAcOKtKsb0snTlVzaGzHR3d2xuFUJ7EFKMmvokD1pZirqASpd68aAA6H8-8SbkcS1D6TjZrxFO_ihBfrqf1R8FhRIggmDFEKlmiwCvvDx-RWLPy182v2Ek8RDAfwoi10KmC8tChYk14UEeKy6pZLS4a2a2CRebHjnKLhKrTFhKXjb_c_TS_H7Q0sEwxqVZpKD3hZ38RvdCmFqbqRzwvNuDLIqjYTvzuw5NaFp-cFc5ErFvVN_bkmEn6m43hLdZwHxTQjfxuiW_zacEm9nV7k8yASZyKDUETWzzveXQLpxLH8Uu6FqTOZlbzHx_l9eBnBeIelq_L59Y71FIWrPBBa2ZShIzmDeQ2gj9LRiuDnOJH2WrI2OQopCpLDg7h27b1L-olyBD7FwjSHyIPdmFMMtXad6vA3Im_2zrOy97py2OIrKRJ41VkSpLNG0o5C3qL5hG9UGSnPgZrK23DXJWZ4i82mK6cufpMAfDMGmWvWXeBUTLF5YYT1sNaA8zUv_0vwYo6vQjBvd48D4aO6fHqIMNgnZG0kvjhK_3ugbScK5KK2MjLJufovVDLDvompTX5ggKpbAoheOiAo0VCSj8eZd7cw09ld2ucsFWaK139c-PSUGBpQH1YvV1caQB84_aYofDlrqepA5K7IvKYpFDApklvGZSqzodZxgY991MkB-v3Ai9cNFirAVAu5Fhz3E-HRSEnT22oPKYAMh4JSLiLdYs_zHG0sc3gw8ncDPyvFbiK2cK3_Aj6R7_uuJacrQQOm1QmKnd9vBKLw3T9gxXsARO3hYhhBtkFAgo7k9YAkiXjgkilSVFjRAIy_SdkFFnyxJw7-kuFOBUTjSvL_HDjl5w3WMxvQhM9mf8GinWnRB6qmyaBtqtIW9-czCtwO8m6l1-50qP2cMB7rAOkn1sp28pgsVYy-fyFy2ldkWFZeVtFYeckFUHNMXl7pxRbharIfm3aQY71MHhR7SkE8YSKbEdbvFhQOfdyeKe9rMYJRgX9G7axRjT4kl1bCJKzSqHJSaNdeFgUV49BdoY_pvqtkVjkGzJUbmbwFBBg97OY6Ux31C7Np2pDiHy-A17sPktnQQn0sHaAZj0xzBPRk6vEiOcS5tDQSqLPPzjnJw75PUaIXzNv8ziDtvuyYF04In1QvSXcQW9im6PJpIh2apTRYhWByYjNcXvN4bZDTxlR5IFZlSXdWb86sEXzAhipYj4T63tdBKNF7ipPjBTrXM0pQH0uts5g6_oBZ-OMxhwcgwvb93xfgkdE59vYRbCzoVZxLKysDrt9dIzTu3ooyMt4_nalAcbOIyvxjLaStL7hurLPJiowQOpH3K8GHr_ZQNWgyA2HD-mUBc940ztwHpY4am1nA7ULxZntGx-n90DzjmAZI1N4nTN9hQPYT9Ps_xGmgvLxhJ40v0yULnos_hn15G9LZVnyNR19JS5IGKjQHlRNjBmNExSpLekLNvIpXegTpMcEZMh_9Wnqm3I5ChIngu-YaQQqUR0PUGG-E10_auOCQYTWr6Y9eoL_8rWtEqIcfLfliAQXjIVbwsQcH1k4Xmw3DvLl1P8TucqOAYXFvdC1OorLa5LTQCrFCVWdWVdVNxY3_ci7GwFuAPUKLzxxBzJ3Erq2QNIZTUJc_XYGdD3EDgx2JSU7eAR5b9Ll17o2LC-qI4_R0-YMkI7y4sFlBVoKxza_1y0)

<details>
  <summary>UML Source</summary>

```
skinparam Style strictuml
skinparam monochrome true
skinparam defaultFontSize 14

note over PlayerLoop: Unity 2019.4 LTS
note over NetworkTicker: InitializeOnLoad
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
