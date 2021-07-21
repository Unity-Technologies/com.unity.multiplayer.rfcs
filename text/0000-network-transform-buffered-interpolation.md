



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
This RFC suggests adding buffering to interpolated NetworkTransforms and suggests an extensible interface for users to use and create different interpolation algorithms.
After this, users should be able to select the interpolation algorithm that best fits the GameObject they want to sync and they should be able to have a default interpolation that resists network jitter.

https://user-images.githubusercontent.com/71790295/126506051-ebfb1974-e4fc-44ca-9923-2be450c722b6.mp4

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
