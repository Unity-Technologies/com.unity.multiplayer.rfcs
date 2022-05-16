# Replace Scene Registration with Scenes in Build List
[feature]: #feature

- Start Date: `2021-07-20`
- RFC PR: [#32](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/32)
- SDK PR: [#1080](https://github.com/Unity-Technologies/com.unity.multiplayer.mlapi/pull/1080)

# Summary
[summary]: #summary

The original NetworkManager Scene registration process was designed to provide users with a means to control which scenes declared within the build settings “scenes in build list” could be loaded during an active session. While this did accomplish the underlying purpose of active session scene loading control, this proposal will outline a new approach that only requires users to populate the “build settings scenes in build list” (as they normally would) while also providing users with the ability to dynamically control which scenes are valid or not based on current game state.

# Motivation
[motivation]: #motivation

The more recent NetworkSceneManager updates allowed for the use of SceneAssets in place of the scene name.  This did help reduce the required user maintenance time by not requiring a user to update the registered scene names within the NetworkManager if they changed the name of a  SceneAsset.  However, this still required users to maintain two lists and the registered scenes approach was a globally static definition that did not allow for a more granular scene loading verification process.    


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

By treating all of the scenes in the build list as registered scenes and providing a scene verification callback handler that was always invoked prior to loading a scene,  users now have one location to register scenes and they can customize the scene verification process.  The new scene verification process provides users, depending upon the user’s custom verification implementation, the ability to associate a specific subset of scenes in the build list based on current game state.  

This approach still required some kind of master registered scenes list in order to provide the very basic scene verification checks of whether the scene name being requested for load (server side) or being commanded to load (client side) was even included in the build.  Leveraging from the SceneUtility.GetScenePathByBuildIndex method, we can create a "Scenes in Build" list every time a NetworkManager is instantiated and Initialized when a Host-Server or client is started.   
  
![](0000-remove-sceneregistration/rem-sceneregistration-1.png)

This completely removes any user registration requirement while also providing a way to quickly validate that the scene requested to be loaded is actually within the scenes in the current build without requiring the user to go beyond the normal steps of populating the Build Settings Scenes in Build list.  

![](0000-remove-sceneregistration/rem-sceneregistration-2.png)

In addition, the NetworkConfig.m_AllowRuntimeSceneChanges property was removed as it no longer applies.  You either have the scenes you are going to use in the build settings “scenes in build list” or you do not.  

![](0000-remove-sceneregistration/rem-sceneregistration-3.png)

User defined scene verification occurs if the user has assigned a callback handler for NetworkSceneManager.VerifySceneBeforeLoading.  The callback is provided with the scene index, scene name, and the mode in which the scene is being loaded.  If the user’s game logic determines it is a valid scene to load then the callback should return true to continue the load scene event process, however if it is not valid the callback should return false and the scene in question will not be loaded.  If the callback is assigned, this process will occur on both server and client(s).

![](0000-remove-sceneregistration/rem-sceneregistration-4.png)


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation
Other than not having to register scenes with the NetworkManager,  there are a few aspects to this update that we should review:
* Because UnityEngine.SceneManagement.SceneManager requires all scenes that can be loaded to be registered in the “scenes in build list”, we no longer need the NetworkConfig.AllowRuntimeSceneChanges property.
* Users need to assign a callback handler to NetworkSceneManager.VerifySceneBeforeLoading in order for pre-loading scene verification to occur.  If nothing is assigned, then all of the “scenes in build list” can be loaded at any time during an active network session.
* Unit tests that load any scenes that are not registered with the “scenes in build list” are required to add their unique scenes to the NetworkSceneManager.ScenesInBuild list.

## **Verifying Scenes** 
Implementing the VerifySceneBeforeLoading delegate is a relatively straightforward approach. Consider the following example:

```C#
using System;
using System.Linq;
using System.Collections.Generic;
using UnityEngine.SceneManagement;
using Unity.Netcode;

[Serializable]
public class ValidGameLevelScene
{
    public string SceneName;
    public LoadSceneMode LoadMode;
}

public class GameLevelSceneVerifyComponent : NetworkBehaviour
{
    public List<ValidGameLevelScene> ValidSceneNamesForCurrentLevel;

    private bool OnVerifySceneBeforeLoading(int sceneIndex, string sceneName, LoadSceneMode loadSceneMode)
    {
        return ValidSceneNamesForCurrentLevel.Select(c => c).Where(c => c.SceneName == sceneName && c.LoadMode == loadSceneMode).Count() > 0;
    }

    public override void OnNetworkSpawn()
    {
        NetworkManager.SceneManager.VerifySceneBeforeLoading = OnVerifySceneBeforeLoading;

        base.OnNetworkSpawn();
    }
}
```
The example above demonstrates how users can quickly create a customized scene verification component that can be used several times in various different scenes throughout a users project.  Each unique instance of the component dictates which scenes a server and/or client can load from the available scenes in the build list.  Each time a new instance is created, the VerifySceneBeforeLoading is reassigned and a new user defined set of “scene loading rules” are loaded relative to the scene containing the GameLevelSceneVerifyComponent instance.


# Drawbacks
[drawbacks]: #drawbacks

* If a user decides to have multiple projects and each project contains a different set of scenes in the “scenes in build list”, then this could become problematic.  However, this really depends upon the user’s requirements and rationale behind this type of configuration.  If it is all for platform specific separation, then it could be recommended to use additive scenes in combination with runtime scene verification that is based on platform type.
* This implementation does not take into account addressables, but that is beyond the scope for this initial effort.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

For the short term, this provides the user with additional dynamic/runtime scene verification capabilities while also removing the need to maintain two sets of scene lists.  The alternative would be to not provide this functionality and stick with the update NetworkSceneManager’s SceneAsset registration process.


# Unresolved questions
[unresolved-questions]: #unresolved-questions

* How could this implementation be modified to handle addresables?
* Should the scene verification process include the client identifier as part of the verification process?  Is there any other information that might be useful for verification purposes?

# Future possibilities
[future-possibilities]: #future-possibilities

This implementation provides a 1:1 relationship between scene names and “scenes in build list” indices values. If/when we decide to derive NetworkSceneManager from the SceneManagerAPI, this will provide us with the ability to handle the broad stroke scene validation when a user decides to load a scene by index value as opposed to scene name.  Additionally, we could allow a user to register SceneGroup registrations.  SceneGroup registrations could include only one base scene (loaded in single mode) and any number of additively loaded scenes or just include only additive scenes.  SceneGroups would allow users to simply specify: “Load this scene group” and all scenes registered within the SceneGroup would be loaded in the same order and mode as they are defined within the SceneGroup.

