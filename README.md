# Unity Multiplayer RFCs

Many changes including bug fixes, documentation improvements and minor features can be implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put through a bit of a design approval process and produce a consensus among the Unity Multiplayer team and the community.

The "RFC" (request for comments) process is intended to provide a consistent and controlled path for new features to enter the Unity Multiplayer, so that both the Unity Multiplayer team and the community can be confident about the direction the project is evolving in.

[Active RFC List →](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pulls)

## When you need to follow this process

You need to follow this process if you intend to make "substantial" changes to Unity Multiplayer projects, documentation or the RFC process itself. What constitutes a "substantial" change is evolving based on community norms and varies depending on what part of the ecosystem you are proposing to change, but may include the following:

- Example A
- Example B
- Example C

Some changes do not require an RFC:

- Example X
- Example Y
- Example Z

If you submit a pull request to implement a new feature without going through the RFC process, it may be closed with a polite request to submit an RFC first.

## Before creating an RFC

A hastily-proposed RFC can hurt its chances of acceptance. Low quality proposals, proposals for previously-rejected features, or those that don't fit into the near-term roadmap, may be quickly rejected, which can be demotivating for the unprepared contributor. Laying some groundwork ahead of the RFC can make the process smoother.

Although there is no single way to prepare for submitting an RFC, it is generally a good idea to pursue feedback from other project developers beforehand, to ascertain that the RFC may be desirable; having a consistent impact on the project requires concerted effort toward consensus-building.

_// todo: forums, discord, slack and other communication channels here._

As a rule of thumb, receiving encouraging feedback from long-standing project developers, and particularly members of the relevant project's team is a good indication that the RFC is worth pursuing.

## What the process is

In short, to get a major feature added to Unity Multiplayer, one must first get the RFC merged into the RFC repository as a markdown file. At that point the RFC is "active" and may be implemented with the goal of eventual inclusion into Unity Multiplayer.

- Fork the RFC repo [RFC repository](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs)
- Copy `0000-template.md` to `text/0000-my-feature.md` (where "my-feature" is descriptive). Don't assign an RFC number yet; This is going to be the PR number and we'll rename the file accordingly if the RFC is accepted.
- Fill in the RFC. Put care into the details: RFCs that do not present convincing motivation, demonstrate lack of understanding of the design's impact, or are disingenuous about the drawbacks or alternatives tend to be poorly-received.
- Submit a pull request. As a pull request the RFC will receive design feedback from the larger community, and the author should be prepared to revise it in response.
- Each pull request will be labeled with the most relevant Unity Multiplayer team, which will lead to its being triaged by that team in a future meeting and assigned to a member of the subteam.
- Build consensus and integrate feedback. RFCs that have broad support are much more likely to make progress than those that don't receive any comments. Feel free to reach out to the RFC assignee in particular to get help identifying stakeholders and obstacles.
- The Unity Multiplayer team will discuss the RFC pull request, as much as possible in the comment thread of the pull request itself. Offline discussion will be summarized on the pull request comment thread.
- RFCs rarely go through this process unchanged, especially as alternatives and drawbacks are shown. You can make edits, big and small, to the RFC to clarify or change the design, but make changes as new commits to the pull request, and leave a comment on the pull request explaining your changes. Specifically, do not squash or rebase commits after they are visible on the pull request.
- At some point, a member of the Unity Multiplayer team will propose a "motion for final comment period" (FCP), along with a *disposition* for the RFC (merge, close, or postpone).
	- This step is taken when enough of the tradeoffs have been discussed that the Unity Multiplayer team is in a position to make a decision. That does not require consensus amongst all participants in the RFC thread (which is usually impossible). However, the argument supporting the disposition on the RFC needs to have already been clearly articulated, and there should not be a strong consensus *against* that position outside of the Unity Multiplayer team. Unity Multiplayer team members use their best judgment in taking this step, and the FCP itself ensures there is ample time and notification for stakeholders to push back if it is made prematurely.
	- For RFCs with lengthy discussion, the motion to FCP is usually preceded by a *summary comment* trying to lay out the current state of the discussion and major tradeoffs/points of disagreement.
	- Before actually entering FCP, *all* members of the Unity Multiplayer team must sign off; this is often the point at which many Unity Multiplayer team members first review the RFC in full depth.
- The FCP lasts ten calendar days, so that it is open for at least 5 business days. This way all stakeholders have a chance to lodge any final objections before a decision is reached.
- In most cases, the FCP period is quiet, and the RFC is either merged or closed. However, sometimes substantial new arguments or ideas are raised, the FCP is canceled, and the RFC goes back into development mode.

## The RFC life-cycle

_// todo_

## Reviewing RFCs

_// todo_

## Implementing an RFC

_// todo_

## RFC Postponement

_// todo_

## Help this is all too informal!

The process is intended to be as lightweight as reasonable for the present circumstances. As usual, we are trying to let the process be driven by consensus and community norms, not impose more structure than necessary.

## Unity Contribution Agreement

All contributions are subject to the [Unity Contribution Agreement ("UCA")](https://unity3d.com/legal/licenses/Unity_Contribution_Agreement). By making a pull request, you are confirming agreement to the terms and conditions of the UCA, including that your Contributions are your original creation and that you have complete right and authority to make your Contributions.

## License

_// todo_

---

**Unity Multiplayer's RFC process owes its inspiration to the [Rust RFC process](https://github.com/rust-lang/rfcs).**
