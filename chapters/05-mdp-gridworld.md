---
layout: chapter
title: "MDPs Part II: Gridworld"
description: Hiking example, softmax agent, stochastic transitions, policies, expected values of possible actions, discounting

---

## MDP Example: Hiking

We introduced Markov Decision Processes (MDPs) with the example of Alice moving around a city with the aim of efficiently finding a good restaurant. Later chapters, which consider agents without full knowledge or complete rationality, will return to this example. This chapter explores some of the components of MDPs that will be important throughout this tutorial. 

We introduce a new sequential decision problem that can be represented by a "gridworld" MDP. Suppose that Bill is hiking. There are two peaks nearby, denoted "West" and "East". The peaks provide different views and Bill must choose between them. South of Bill's starting position is a steep hill. Falling down the hill would result in painful (but non-fatal) injury and end the hike early.

We represent Bill's hiking problem with a gridworld similar to Alice's restaurant choice example. The peaks are terminal states, providing differing utilities. The steep hill is also a terminal state, with negative utility. We assume a negative, constant time-cost -- so Bill prefers a shorter hike. 

~~~~
var noiseProb = 0;
var alpha = 100;
var utilityEast = 10;
var utilityWest = 1;
var utilityHill = -10;
var timeCost = -.1;

var params = makeHike(noiseProb, alpha, utilityEast, utilityWest, utilityHill, timeCost);
var startState = [0,1];
displayGrid(params, startState);
~~~~

We start with a *deterministic* transition function. This means that Bill's only risk of falling down the steep hill is due to softmax noise in his actions. With `alpha=100`, the chance of this is tiny, and so Bill will take the short route to the peaks. The agent model is the same as the end of [Chapter 4]('/chapters/04-mdp.md'). The function `mdpSimulate` is also a library function and we will re-use throughout this chapter. 


~~~~
var mdpSimulate = function(startState, totalTime, params, numRejectionSamples){
  var alpha = params.alpha;
  var transition = params.transition;
  var utility = params.utility;
  var actions = params.actions;
  var isTerminal = function(state){return state[0]=='dead';};


  var agent = dp.cache(function(state, timeLeft){
    return Enumerate(function(){
      var action = uniformDraw(actions);
      var eu = expUtility(state, action, timeLeft);    
      factor( alpha * eu);
    return action;
    });      
  });
  
  
  var expUtility = dp.cache(function(state, action, timeLeft){
    var u = utility(state,action);
    var newTimeLeft = timeLeft - 1;
    
    if (newTimeLeft == 0 | isTerminal(state)){
      return u; 
    } else {                     
      return u + expectation( Enumerate(function(){
        var nextState = transition(state, action); 
        var nextAction = sample(agent(nextState, newTimeLeft));
        return expUtility(nextState, nextAction, newTimeLeft);  
      }));
    }                      
  });
  
  var simulate = function(startState, totalTime){
  
    var sampleSequence = function(state, timeLeft){
      if (timeLeft == 0 | isTerminal(state)){
        return [];
      } else {
      var action = sample(agent(state, timeLeft));
        var nextState = transition(state,action); 
        return [[state,action]].concat( sampleSequence(nextState,timeLeft-1 ))
      }
      };
    // repeatedly sample trajectories for the agent and return ERP
    return Rejection(function(){return sampleSequence(startState, totalTime);}, numRejectionSamples);
  };

  return simulate(startState, totalTime);
};

// parameters for building Hiking MDP
var utilityEast = 10;
var utilityWest = 1;
var utilityHill = -10;
var timeCost = -.1;
var startState = [0,1];

// parameters for noisy agent (but no transition noise)
var alpha = 100;
var noiseProb = 0;
var params = makeHike(noiseProb, alpha, utilityEast, 
    utilityWest, utilityHill, timeCost);
displayGrid(params,startState);

var totalTime = 12;
var numRejectionSamples = 1;
var out = sample( mdpSimulateTemp(startState, totalTime, params, 
    numRejectionSamples) );
displaySequence( out, params);

~~~~

## Hiking under the influence 
If we set the softmax noise parameter `alpha=10`, the agent will often take sub-optimal decisions. While not realistic in Bill's situation, this might better describe an intoxicated agent. Since the agent is noisy, we sample many trajectories in order to approximate the full distribution. To construct an ERP based on these samples, we use the built-in function `Rejection` which performs inference by rejection sampling. (Our goal here is not inference over trajectories and so the `Rejection` function does not need any `factor` statement in its body). In this case, when the agent takes a suboptimal action, it will take *longer* than five steps to reach the East summit. So we plot the distribution on the length of trajectories to summarize the agent's behavior. 


~~~~
// parameters for building Hiking MDP
var utilityEast = 10;
var utilityWest = 1;
var utilityHill = -10;
var timeCost = -.1;
var startState = [0,1];

var alpha = 10;
var noiseProb = 0;
var params = makeHike(noiseProb, alpha, utilityEast,
    utilityWest, utilityHill, timeCost);

var totalTime = 12;
var numRejectionSamples = 500;
var erp = Enumerate( function(){
  return sample( 
    mdpSimulateTemp(startState, totalTime, params, numRejectionSamples)).length;
});

viz.print(erp);


~~~~

### Exercise
Sample some of this noisy agent's trajectories. Does it ever fall down the hill? Why not? By varying the noise and other parameters, find a setting where the agent's modal trajectory length is the same as the `totalTime`.
<!-- let alpha=0.5 and action cost = -.01 -->


## Hiking with stochastic transitions

When softmax noise is high, the agent will make many small "mistakes" (i.e. suboptimal actions) but few large mistakes. In contrast, sources of noise in the environment will change the agent's state transitions independent of the agent's preferences. In the hiking example, imagine that the weather is very wet and windy. As a result, Bill will sometimes intend to go one way but actually go another way (because he slips in the mud). In this case, the shorter route to the peaks might be too risky for Bill. We will use a simplified model of bad weather. We assume that at every state and time, there is a constant independent probability `noiseProb` of the agent not going in their chosen direction. The independence assumption is unrealistic (since if a location is slippery at one timestep it's more likely slippery the next) but is simple and conforms to the Markov assumption for MDPs. When the agent does not go in their intended direction, they randomly go in one of the two orthogonal directions.

We set `noiseProb=0.1` and show that (a) the agent's first intended action is "up" not "right", and (b) that the agent's trajectory lengths vary due to stochastic transitions. *Exercise:* keeping `noiseProb=0.1`, find settings for the arguments to `makeHike` such that the agent goes "right" instead of up. <!-- put up action cost to -.5 or so --> 

~~~~
// parameters for building Hiking MDP
var utilityEast = 10;
var utilityWest = 1;
var utilityHill = -10;
var timeCost = -.1;
var startState = [0,1];

var alpha = 100;
var noiseProb = 0.1;
var totalTime = 12;
var numRejectionSamples = 1;
var params = makeHike(noiseProb, alpha, utilityEast, utilityWest, utilityHill, timeCost);
var out = sample(mdpSimulateTemp(startState, totalTime, params, numRejectionSamples));
displaySequence(out,params);
print('stochastic transitions: ' + out);
~~~~

In a world with stochastic transitions, the agent sometimes finds itself in a state it did not intend to reach. The functions `agent` and `expUtility` (inside `mdpSimulate`) implicitly compute the expected utility of actions for every possible future state -- including states that the agent will try to avoid. In the MDP literature, this function from state-time pairs to actions (or distributions on actions) is called a *policy*. [Sidenote: For infinite horizon MDPs, policies are functions from states to actions, which makes them somewhat simpler to think about.]. 

The example above showed that the agent chooses the long route (avoiding the risky hill). At the same time, the agent computes what to do if it ends up moving right on the first action.

~~~~
// parameters for building Hiking MDP
var utilityEast = 10;
var utilityWest = 1;
var utilityHill = -10;
var timeCost = -.1;

// Change start state from [1,0] to [1,1] and reduce time
var startState = [1,1];
var totalTime = 12 - 1;
var numRejectionSamples = 1;

var alpha = 100;
var noiseProb = 0.1;
var params = makeHike(noiseProb, alpha, utilityEast, utilityWest, utilityHill, timeCost);
var out = sample(mdpSimulateTemp(startState, totalTime, params, numRejectionSamples));
displaySequence(out,params);
print('stochastic transitions: ' + out);
~~~~

Extending this idea, we can output and visualize the expected values of actions the agent *could have* on a trajectory. For each state in a trajectory, we compute the expected value of each possible action (given the state and the time remaining). The resulting numbers are analogous to Q-values in infinite-horizon MDPs. 

The expected values we seek to display are already being computed: we add a function addition to `mdpSimulate` in order to output them. 


~~~~

var mdpSimulate = function(startState, totalTime, params, numRejectionSamples){
  var alpha = params.alpha;
  var transition = params.transition;
  var utility = params.utility;
  var actions = params.actions;
  var isTerminal = function(state){return state[0]=='dead';};

    
  var agent = dp.cache(function(state, timeLeft){
    return Enumerate(function(){
        var action = uniformDraw(actions);
        var eu = expUtility(state, action, timeLeft);    
        factor( alpha * eu);
        return action;
        });      
    });
  
  
  var expUtility = dp.cache(function(state, action, timeLeft){
    var u = utility(state,action);
    var newTimeLeft = timeLeft - 1;
    
    if (newTimeLeft == 0 | isTerminal(state)){
      return u; 
    } else {                     
      return u + expectation( Enumerate(function(){
        var nextState = transition(state, action); 
        var nextAction = sample(agent(nextState, newTimeLeft));
        return expUtility(nextState, nextAction, newTimeLeft);  
      }));
    }                      
  });
  
  var simulate = function(startState, totalTime){
  
    var sampleSequence = function(state, timeLeft){
      if (timeLeft == 0 | isTerminal(state)){
        return [];
      } else {
      var action = sample(agent(state, timeLeft));
        var nextState = transition(state,action); 
        return [[state,action]].concat( sampleSequence(nextState,timeLeft-1 ))
      }
    };
    return Rejection(function(){return sampleSequence(startState, totalTime);}, numRejectionSamples);
  };

 // Additions for outputting expected values
  var unzip = function(ar,i){
    return map(function(tuple){return tuple[i];}, ar);
  };
  
  var downToOne = function(n){
    return n==0 ? [] : [n].concat(downToOne(n-1));
  };
  
  var getExpUtility = function(){
    var erp = simulate(startState, totalTime);
    var states = unzip(erp.MAP().val,0); // go from [[state,action]] to [state]
    var timeStates = zip(downToOne(states.length), states); // [ [timeLeft, state] ] for states in trajectory

    // compute expUtility for each pair of form [timeLeft,state]
    return map( function(timeState){
      var timeLeft = timeState[0];
      var state = timeState[1];
      return [JSON.stringify(state), map(function(action){
        return expUtility(state, action, timeLeft);
      }, params.actions)];
    }, timeStates);
  };
  
  // mdpSimulate now returns both an ERP over trajectories and
  // the expUtility values for MAP trajectory
  return {erp: simulate(startState, totalTime),
          stateToExpUtilityLRUD:  getExpUtility()};

};

// modify the utilities so that West is better than previously
var utilityWest = 7;
var utilityEast = 10;
var utilityHill = -10;
var timeCost = -.2;
var startState = [0,1];

var alpha = 100;
var noiseProb = 0.05;
var totalTime = 12;
var numRejectionSamples = 1;
var params = makeHike(noiseProb, alpha, utilityEast, 
    utilityWest, utilityHill, timeCost);

var out = mdpSimulate(startState, totalTime, params, numRejectionSamples);
var trajectory = sample(out.erp);
var expUtilityValues = out.stateToExpUtilityLRUD;
print(expUtilityValues);
// display both the trajectory and the expUtilityValues

~~~~





decrease time: more time pressure moves you to shortcut. (as does increasing the action cost). decrease time enough and you just go to close hill (sample for increasing action cost). 

