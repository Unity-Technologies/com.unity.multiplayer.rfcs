# Network Transform Buffered Interpolation
[feature]: #feature

- Start Date: `2021-07-19`
- RFC PR: [#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- SDK PR: [#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/pull/0000)

# Summary
[summary]: #summary

This RFC proposes upgraded interpolation options from the old NetworkTransform's interpolation that's resistent to jitter and more flexible for users.


# Motivation
[motivation]: #motivation


NetworkTransform needs a way to help users smooth movements that can resist to different network conditions and allows users to extend it if needed.
The previous implementation didn't offer much flexibility on what interpolation algorithm to use. If I wanted to create a Kalman Filter interpolator or a Quadratic interpolator instead of a linear interpolator, I would need to reimplement my own NetworkTransform.

It also didn't take into account network jitter, which becomes a visible issue on overloaded or unstable networks like you can find with mobile platforms 

The following video shows the MLAPI-0.1.0 NetworkTransform under 5% packet loss and 60-100ms jitter.
https://user-images.githubusercontent.com/71790295/126506051-ebfb1974-e4fc-44ca-9923-2be450c722b6.mp4

![](0000-network-transform-buffered-interpolation/jitter.jpg)


This RFC suggests an extensible interface for users to use and create different interpolation algorithms and adding a default buffered interpolator that'll resist to jitter. 
After this, users should be able to select the interpolation algorithm that best fits the GameObject they want to sync.

In addition, this RFC suggests removing the 
```C#
        public AnimationCurve DistanceSendrate = AnimationCurve.Constant(0, 500, 20);
```
configuration (that was used to set interpolation times) to use instead a swappable interpolator.



Note this RFC only intends to touch "ghost" interpolation. It'll still be up to users to interpolate their authoritative objects however they want. If for example Client 1 is authoritative over a player object, this RFC will add interpolation to the server object and the other clients' objects, not Client 1's object. It'll still be up to users to interpolate that authoritative object (using RigidBody's interpolation for example or anything else).



# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

NetworkTransform will now have this new configuration:
- Interpolator: Allows users to select the interpolation algorithm they want to use.

It'll be up to the interpolator to offer its own configuration.
 
An `IInterpolator` interface has been added, to allow users to create their own custom interpolator.
The implemented interpolator needs to keep its own state and will be given new values each time ones are available to the NetworkTransform's OnValueChanged.

```C#
public interface IInterpolator<T>
{
    public void Update(float deltaTime);
    public void NetworkTickUpdate(float fixedDeltaTime);
    public void AddMeasurement(T newMeasurement, int sentTick);
    public T GetInterpolatedValue();
    public void Teleport(T value, int sentTick);
}
```

```C#
public class SimpleInterpolator : IInterpolator<Vector3>
{
    // a user's simple interpolator
    public void Update(float deltaTime)
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

    public void Teleport(Vector3 value)
    {
        m_UpdatedVector = value;
        m_StartVector = value;
        m_EndVector = value;
        m_CurrentTime = 1;
    }

    public void NetworkTickUpdate(float fixedDeltaTime) {}
}
```
A factory is used to select and configure which interpolator to use for the NetworkTransform.
```C#
public abstract class InterpolatorFactory<T> : ScriptableObject
{
    public const string BaseMenuName = "MLAPI/Interpolator/";
    public abstract IInterpolator<T> CreateInterpolator();
}

public abstract class BufferedLinearInterpolatorFactory<T> : InterpolatorFactory<T>
{
    [SerializeField]
    public float InterpolationTime = 0.100f;
}

[CreateAssetMenu(fileName = "BufferedLinearInterpolatorVector3", menuName = BaseMenuName + "BufferedLinearInterpolatorVector3", order = 1)]
public class BufferedLinearInterpolatorVector3Factory : BufferedLinearInterpolatorFactory<Vector3>
{
    public override IInterpolator<Vector3> CreateInterpolator()
    {
        return new BufferedLinearInterpolatorVector3(this);
    }
}
```

Settings for each types of interpolators (like the above `InterpolationTime`) are owned by the interpolator's factory. NetworkTransform allows selecting which interpolator factory to use.
![](0000-network-transform-buffered-interpolation/InterpolatorConfiguration.png)

![](0000-network-transform-buffered-interpolation/InterpolatorConfigurationSO.png)

A default `BufferedLinearInterpolator` is provided. The buffered linear interpolator will buffer values before making them available to NetworkTransform's value update. This will allow NetworkTransform to accumulate jittered network values without affecting the transform's state and latter be able to consume them at regular intervals.

The buffered interpolator will maintain a buffered list and use the global Network Time's buffer configuration to determine how long to keep values in buffer.

![](0000-network-transform-buffered-interpolation/jitter_with_buffer.jpg)

It is advised to use the same interpolator for all your NetworkTransforms. This way, you're making sure all your objects will be synchronized and use the same delays. Having an interpolator with 100ms delay and another with 500ms delay could cause visible overlaps and desyncs where an object at time say 10s tries to interact with an object buffered at time 9.6s.

If you have a case where you'd like to use a different configuration, you can create a new instance of the interpolator scriptable object and assign it to your object.

If users don't want interpolation, they can use the `NoInterpolation` interpolator which will take in new values and present them directly when asked, without doing anything else.

- Left: Host with 0.1.0 NetworkTransform and new buffered NetworkTransform
- Right: Client with 60-100ms jitter and 5% packet loss
- https://user-images.githubusercontent.com/71790295/126731436-f9e89d75-4973-49e8-bbb5-84ff67c48a9b.mov


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Interpolator integration with NetworkTransform
Users can select their interpolator using a global configuration set in a scriptable object.
```C#
[SerializeField]
private InterpolatorFactory<Vector3> m_PositionInterpolator;
```
On Awake, NetworkTransform will create its own instance to use.
```C#
private void Awake()
{
    PositionInterpolator = m_PositionInterpolatorFactory.CreateInterpolator();
```
That interpolator can then be used in NetworkTransform.
```C#
private void OnNetworkStateChanged(NetworkState oldState, NetworkState newState)
{
    // ...
    PositionInterpolator.AddMeasurement(newState.Position);
}
```

The authoritative value coming from the network is now intercepted by the interpolator. The value applied to the transform is now the interpolated value instead.

```C#
private void ApplyNetworkState(NetworkState netState)
{
    // ...
    m_Transform.position = PositionInterpolator.GetInterpolatedValue(); // this doesn't update the value, just queries it

```

The interpolated value is updated every frame, only if the game is not allowed to update the transform. (For example with a server driven transform, only the client would be interpolated)

```C#
private void Update()
{
    // ...

    if (!CanUpdateTransform)
    {
        PositionInterpolator.Update(Time.deltaTime);
        ApplyNetworkState(m_NetworkState.Value);
    }
}
```

## Buffered Interpolator

The buffered interpolator will maintain a list of buffered items, associated with the tick it was sent. This means the NetworkTransform, in addition to sending the state to other nodes will also need to include the tick in its sent state.


```C#
public void AddMeasurement(T newMeasurement, int SentTick)
{
    m_Buffer.Add(new BufferedItem<T>() {item = newMeasurement, tickSent = SentTick});
}
```
Every tick, the interpolator will make a new buffered item available according to how much time it should spend in the buffer.
```C#
private double ServerTickBeingHandledForBuffering => NetworkManager.Singleton.ServerTime.Tick; // override this if you want configurable buffering, right now using ServerTick's own global buffering

private void TryConsumeFromBuffer()
{
    for (int i = m_Buffer.Count - 1; i >= 0; i--)
    {
        var bufferedValue = m_Buffer[i];
        if (bufferedValue.tickSent <= ServerTickBeingHandledForBuffering)
        {
            m_LerpStartValue = m_LerpEndValue;
            m_LerpEndValue = bufferedValue.item;
            m_ValueLastTick = bufferedValue.tickSent;
            m_Buffer.RemoveAt(i);
        }
    }
}
```

# Drawbacks
[drawbacks]: #drawbacks

Current plan is to offer less configuration than the previous NetworkTransform interpolation (by only using a float to specify the amount of time to interpolate vs the old AnimationCurve which allowed to specify the amount of time to interpolate varying over the distance to interpolate).

Current interpolation doesn't make any assumptions on future rollback needs.
This means that any future rollback code we use that wants to correct interpolated values might require new explicit APIs that would break user defined implementations (users would need to implement any new method).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Giving the option to have different buffer length per network transform
Current configuration is to have everything buffered with the same value. This way, you promote better world consistency, since everyone is on the same timeline.
However, one might want to have exceptions where you don't care about smoothness for your transform and only want the quickest position update, as soon as you have it and not care about jittery state changes. These objects would then need to have little or no buffer, contrary to the global setting in the time system.
At this point, users could either create their own interpolator inheriting from buffered interpolator if they still want smaller buffering
```C#
    public class BufferedLinearInterpolator<T> : IInterpolator<T> where T : struct
    {
        // ...
        private double ServerTickBeingHandledForBuffering => ServerTick; // override this if you want configurable buffering, right now using ServerTick's own global buffering
```
Or they could use `NoInterpolator` which just returns your latest position, without doing anything to it.

## Other ways to reduce jitter

Reducing the number of packets sent is a good way to prevent jitter, however the goal of this RFC is really to mitigate the jitter that does happen, even after mitigation.
Increasing send rate will help reducing the buffer size needed for clean interpolation. With shorter ticks, you need to wait a shorter time before saying you can process a server position. This could be done through education when talking about these issues.

## Other design
The interpolator could have been hard coded in NetworkTransform without actually being a separate class. This could have made NetworkTransform more self contained, but would also have been more cumbersome to extend our interpolation. With .meta file default values, you can set a ScriptableObject as a default for your class, no extra user action should be needed to use NetworkTransform with default values.

## Impact of not doing this
Buffered interpolation is simple on paper, but can be frustrating to implement, even for netcode experts. It's a recurring technique to allow resisting to bad network conditions like you can find on mobile platforms.
Having this set as a default will allow giving users a NetworkTransform that is smooth by default, on a wide range of network conditions.

## Why in NetworkTransform and not NetworkVariables

Doing it directly in netvars right now isn't super useful and would bind our API for nothing. Without snapshot, there doesn't seem to be a lot of good usecases for interpolation in netvars, so let's implement this in NetworkTransform until we're more defined in NetVars.
Plus, since we're using a custom state struct in NetworkTransform, it's work that'll be needed anyway there.

With NetworkTransform we control when the state updates and when state consumptions are executed (on tick), which we do not with NetVars, not unless we restrict the API greatly.

## Rollback/reconciliation and interpolator.
Current interpolator only affects ghosts and doesn't affect local objects. An eventual predicted player wouldn't be affected by this interpolation.
Future prediction might want to interpolate corrected values instead of teleporting players. This should be handled by prediction code and shouldn't touch this interpolator.

## Other APIs for interpolator
The Interpolator's `Update()` method could have been injected with the last 3 positions. 
```C#
Vector3 Update(Vector3 pos1, Vector3 pos2, Vector3 pos3)
{
    // interpolate over those 3 positions, even if they are rolledback positions
    return interpolatedValue;
}
```
This assumes the number of values needed for interpolator. What if your interpolator needs 4 values? or just 2?. With `AddMeasurement`, you're allowing more flexibility for user implementations. However you require your interpolator to track state, while state injection in the above example would centralize state in the snapshot system. Having the interpolator track its own state could cause desync issues in the future, however offering a `Teleport` method can serve as a "reset" button for interpolators.
```C#
void RewriteHistory(Vector3[] newPositions, int[] newTicks)
{
    m_Interpolator.Teleport(newPositions[0], newTicks[0]);
    for(int i=1; i<newPositions.Count; i++)
    {
        m_Inteprolator.AddMeasurement(newPositions[i], newTicks[i]);
    }
}
```

# Prior art
[prior-art]: #prior-art

Gafferongames suggests different solutions for buffered interpolation, according to your needs.
https://gafferongames.com/post/snapshot_interpolation/
It talks about Hermite interpolation and mentions how linear interpolation created wobbly artefacts on child cubes attached to a rotating cube.
The article does mention Hermite interpolation requires more bandwidth (is it needs to sync velocity in addition to transform state).
Users wanting to use this could extend NetworkTransform to sync velocity and implement their own interpolator.

## HLAPI
HLAPI's interpolation required a Rigidbody being attached to the transform (and was setting that RB's velocity accordingly). This is a simple solution that works for certain cases.
If a user wants to network any non-physics objects or non-kinematic objects, they would need to implement their own interpolation, which can be frustrating to implement, even for netcode experts.
Here's forum post on these frustrations
https://forum.unity.com/threads/networktransform-interpolation.335155/

## Unreal
TODO

## Mirror
TODO

## Photon
TODO

/////////////////////////////

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

- For framework, tools, and library proposals: Does this feature exist in other networking stacks and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other projects, provide readers of your RFC with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other projects.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC. Please also take into consideration that Unity Multiplayer sometimes intentionally diverges from common multiplayer networking features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## What I expect to resolve through the RFC process before this gets merged
## What I expect to resolve through the implementation of this feature before stabilization
- How interpolation configuration is exposed to users would be solved during stabilization

/

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Sam: Kalman Filter Extrapolation, quadratic interpolation, Hermite Interpolation


Think about what the natural extension and evolution of your proposal would be and how it would affect the Unity Multiplayer as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the Unity Multiplayer in your proposal. Also consider how the this all fits into the roadmap for the project and the team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.
