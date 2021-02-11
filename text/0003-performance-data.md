- Feature Name: Network Tick and Profiler Notification System
- Start Date: 1-29-2021
- RFC PR: [Unity-Technologies/com.unity.multiplayer.rfcs#0000](https://github.com/Unity-Technologies/com.unity.multiplayer.rfcs/pull/0000)
- Issue: [Unity-Technologies/com.unity.multiplayer#0000](https://github.com/Unity-Technologies/com.unity.multiplayer/issues/0000)

# Summary
[summary]: #summary

Currently there is Transport performance data and MLAPI performance data that is not being propagated up externally for any consumer to render or consume. The use cases for this would be useful for technologies such as a **Network Stats Overlay** or a **Profiler** (**Network Messages** and **Network Operation** need this data in the vanilla Unity Profiler). Users need a viable and low overhead way to obtain this data (such as RPCs sent this frame/tick) from MLAPI and utilize this data in the way that they want for their own metrics.

# Motivation
[motivation]: #motivation

This is needed to provide both a default and user specific implementations for future Network Profilers and Network Stats overlays. The MLAPI source code can remain Open Source, and yet, the data being streamed can be hooked up to external and low level systems.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In order to allow first and third party rendering of the data we will need to allow both MLAPI and its transports to propagate data up the chain. 
If the profiler flags are enabled the following will need to happen: 
Transports will need to output the number of bytes being sent in the current tick and the latency. The data will be written out to a ProfilerTickData chunk. This data will be generic and transport agnostic.
MLAPI will need to output the number of networked objects and the amount of RPCs being sent. This data will be added to the ProfilerTickData chunk.

External Tools and APIs will be able to then get this fully written ProfilerTickData at the end of the frame for them to do what they wish (Render to Profiler Stats Overlay or Display/Record in a Profiler or file)

As this data is largely a Dictionary, accessing its data will be as simple as accessing things by specified fields. Additionally the rendering or logging of these statistics will fall on the user and not affect the Tick or be done at all within the MLAPI framework.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation


In order to limit the interface, we are pushing the collected set or performance data to be obtainable only at the end of a tick. The following is roughly what this object could look like. Currently this object will be updated in ```NetworkManager.LateUpdate()``` for the MLAPI and ```Transport.Send()``` and other relevant transport method for the transport layer. At the end of the tick the user will be notified that the relevant data is complete and ready (both the MLAPI data and Transport data). Since MLAPI is responsible for collecting the data, and the data from transports will be generic and apply to all/most transports, MLAPI as  the owner of the collection of performance data will call ```GetTransportGetData()``` on the transports through an interface in order to get the data exactly when it needs it in the pipeline before propagating the data out externally. Whether or not the data is collected per transport can also be toggled by a runtime flag ‘profilerEnabled’.

```cs
namespace MLAPI {
	class ProfilerConstants {
		public const string NumberOfServerRPCs = "numberOfServerRPCs";

		public const string TickDuration = "tickDuration";
		public const string NumberOfBuffMessagesOut = "numberOfBuffMessagesOut";
		public const string NumberOfBuffMessagesIn = "numberOfBuffMessagesIn";
		public const string NumberOfUnBuffMessagesOut = "numberOfUnBuffMessagesOut";
		public const string NumberOfUnBuffMessagesIn = "numberOfUnBuffMessagesIn";
		public const string NumberBytesSent = "numberBytesSent";
        public const string NumberBytesReceived = "numberBytesReceived";
	}

	public class ProfilerTickData {
		public int tickID;

		public readonly Dictionary<string, int> tickData = new Dictionary<string, int>();
	}

	public interface ITransportProfilerData {
		Dictionary<string, int> GetTransportGetData();
	}
}

namespace MLAPI {
    class NetworkManager{
		
		private static int tickID = 0;
		public delegate void PerfomanceDataEventHandler(ProfilerTickData profilerData);
		event PerfomanceDataEventHandler PerfomanceDataEvent;
         
		void LateUpdate() {
             ...
             ProfilerTickData profilerData = new ProfilerTickData();
             profilerData.tickID = tickID++;
             ...
             // Transports do their send and receives
             ...
             profilerData.tickData[ProfilerConstants.NumberOfServerRPCs] += 1;
             ...            
			 var profileTransport = NetworkConfig.NetworkTransport as ITransportProfilerData;
			 if(profileTransport != null){
					var transportProfilerData = profileTransport.GetTransportGetData();
					transportProfilerData.ForEach(x => { if (!profilerData.tickData.ContainsKey(x.Key)) profilerData.tickData.Add(x.Key, x.Value); });
			  }
             PerfomanceDataEvent?.Invoke(profilerData);
         }
    }
}


namespace MLAPI.Transport {
    class UnetTransport: Transport, ITransportProfilerData {
         
        private static Dictionary<string, int> transportProfilerData;
        private static bool profilerEnabled;

        void Init(){
             transportProfilerData = new Dictionary<string, int>();
        }
         
         Dictionary<string, int> GetTransportGetData() {
             return transportProfilerData;
         }
         
         void Send(ulong clientId, ArraySegment<byte> data, string channelName){
             if(profilerEnabled){
                  transportProfilerData[ProfilerConstants.NumberOfBuffMessagesOut] += 1;
             }
         }
    }
}

namespace ThirdParty {
    class StatsRenderer{
        void Start(){
            NetworkManager.instance.PerfomanceDataEvent += OnPerformanceEvent;
        }
        
        void OnPerformanceEvent(ProfilerTickData profilerData){
            IMGUI.textlabel("Num Server RPCs", profilerData.tickData[ProfilerConstants.NumberOfServerRPCs]);
        }
    }
}

```

# Drawbacks
[drawbacks]: #drawbacks

With a dictionary we occur the performance overhead of boxing somewhat. Additionally we require our callers to look up the appropriate types for the data that they are pulling out of the dictionary.
However, changes to the ProfilerTickData will not cause large scale changes to the transport, MLAPI, and all third party consumers.
Each transport is also required to implement any required changes needed to fill out the relevant transport-level information. Of course not filling out the fields simply means the Overlay, Profiler, Logger will just display default values for the field types. This means they can implement after an initial release of their transport, if need be.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design of single memory block is the best for locality of data. 
There is less room for user error since there is only one readonly object that they get access to.
There is no room for user injected data or exceptions since we aren’t using an event signaling or actions. All user interaction for rendering happens at end of tick so they can’t affect the time of profiling or cause unforeseen issues.

The impact of not doing a Network Tick and Profiler Notification System is that we’d loose the Profiling capabilities that existed in the vanilla Profiler with Mirror/UNET.

# Prior art
[prior-art]: #prior-art

Writing all pertinent data per tick is a similar method to what most rendering pipelines do, which is gather relevant data store in a buffer and render it. Since Multiplayer requires the same effective speed as our Render Pipelines we want to make sure it is as stable and robust as possible.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Some of the design involving profilerEnabled flag and using an interface is based upon the MultPlex Transport which may use other transports underneath. Is this truly necessary?

# Future possibilities
[future-possibilities]: #future-possibilities

This is a low impact and visibility change. All in all, it's unlikely that there would be much in future possibilities or requests to this feature.
