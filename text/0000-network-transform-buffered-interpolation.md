



# Network Transform Buffered Interpolation
[feature]: #feature

- Start Date: `2021-07-19`
- RFC PR: [#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- SDK PR: [#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/pull/0000)

# Summary
[summary]: #summary

This RFC proposes upgraded interpolation options from the old NetworkTransform's interpolation that's resistent to jitter and clearer for users.




# Motivation
[motivation]: #motivation


NetworkTransform needs a way to help users smooth movements that can resist to different network conditions.
The previous implementation handled interpolation but didn't take into account network jitter. It also didn't offer much flexibility on what interpolation algorithm to use.

![](0000-network-transform-buffered-interpolation/jitter.jpg)


This RFC suggests adding buffering to interpolated NetworkTransforms and suggests an extensible interface for users to use and create different interpolation algorithms.
After this, users should be able to select the interpolation algorithm that best fits the GameObject they want to sync and they should be able to have a default interpolation that resists network jitter.

![](0000-network-transform-buffered-interpolation/jitter_with_buffer.jpg)


In addition, this RFC suggests removing the 
```cs
        public AnimationCurve DistanceSendrate = AnimationCurve.Constant(0, 500, 20);
```
Property (that was used to set interpolation times) to use instead a simple number, which would be less confusing for users.

The following video shows the MLAPI-0.1.0 NetworkTransform under 5% packet loss and 60-100ms jitter.
https://user-images.githubusercontent.com/71790295/126506051-ebfb1974-e4fc-44ca-9923-2be450c722b6.mp4

Note this RFC only intends to touch "ghost" interpolation. It'll still be up to users to interpolate their authoritative objects however they want.
If for example I have Client 1 authoritative of a player object, this RFC will add interpolation to the server object and the other clients' objects, not Client 1's object. It'll still be up to users to interpolate that authoritative object (using RigidBody's interpolation for example, or anything else).

TODO
Doing it directly in netvars right now isn't super useful and would bind our API for nothing. Since there's not a lot of good usecases for interpolation in netvars without snapshots, let's implement this in NetworkTransform until we're more defined in NetVars. 
rollback/correction event eventually, not now since we don't have any concrete plans for reconciliation APIs. This means a breaking change in the future for everyone that creates custom interpolators, where they'll need to handle an eventual "IsRollingBack" event. Worst comes to worst, we could even have a flag in an eventual prediction system that turns on when you're rolling back and allow interpolators to query it, without being injected with it. 
could have done 3 values from snapshot in "interpolate" method. This assumes the number of values needed for interpolator. With AddMeasurement, you're allowing more flexibility. However you require your interpolator to track state. Having a reset state event allows to trigger this for future snapshot rollback.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

NetworkTransform will now have these two configurations:
- Max interpolation time: If set to 200ms, the transform would take 200ms to reach its target position from its current position. (I'm using "position" here, but this applies to rotation and scale as well)
- Interpolator: Allows users to select the interpolation algorithm they want to use. 
 
An `IInterpolator` interface has been added, to allow users to create their own custom interpolator.
The implemented interpolator needs to keep its own state and will be given new values each time ones are available to the NetworkTransform's OnValueChanged.

```cs
public interface IInterpolator<T>
{
    public void InterpolationUpdate(float deltaTime);
    public void AddMeasurement(T newMeasurement);
    public T GetInterpolatedValue();
}
```
Interpolators would then have to implement their own stateful version

```cs
    // a user's simple interpolator
    public void InterpolationUpdate(float deltaTime)
    {
        m_CurrentTime += deltaTime;
        m_UpdatedVector = Vector3.Lerp(m_StartVector, m_EndVector, m_CurrentTime / MaxLerpTime);
    }

    public void AddMeasurement(Vector3 newMeasurement)
    {
        m_EndVector = newMeasurement;
        m_CurrentTime = 0;
        m_StartVector = m_UpdatedVector;
    }

    public Vector3 GetInterpolatedValue()
    {
        return m_UpdatedVector;
    }
```

Settings for each types of interpolators (like the above `MaxLerpTime`) are exposed in NetworkTransform's inspector, in addition to having a toggle to select which interpolator to use.

A default `BufferedLinearInterpolator` is provided. The buffered linear interpolator will buffer values before making them available to NetworkTransform's value update. This will allow NetworkTransform to accumulate jittered network values without affecting the transform's state and latter be able to consume them at regular intervals.

If users don't want interpolation, they can use the `NoInterpolator` which will take in new values and present them directly when asked, without doing anything else.

How users set these which interpolator would be an implementation detail. A factory ScriptableObject could be used or a custom inspector UI could be developed for this.



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

# Drawbacks
[drawbacks]: #drawbacks

Why should we _not_ do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Sam notes:
Reducing number of packets sent is a good way to prevent jitter, however the goal of this RFC is really to mitigate the jitter that does happen, even after mitigation.
Increasing send rate will help reducing the buffer size needed for clean interpolation. With shorter ticks, you need to wait a shorter time before saying you can process a server position.
Other designs: variable buffer todo
impact of not doing this: it's a pretty common pattern that comes back often. Having users implement it themselves is not great.

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Sam:
https://gafferongames.com/post/snapshot_interpolation/

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
