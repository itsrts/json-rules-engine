let tagconditions = {
  'type': 'tag', // generated by tableid
  'tag': '11111111-2222-3333-4444-123456789012', //persisted
  'conditions': { // generated
     "fact": "tags",
     "operator": "in",
     "value": '11111111-2222-3333-4444-123456789012'
   }
}
let eligconditions =  {
  'type': 'eligibility',
  'equationId': '11111111-2222-3333-4444-5678901234567',
  'eligibilityId': '11111111-2222-3333-4444-123456789012',
  'fieldid': 'age'
}

let JsonRules = require('json-rules')
let achievementRules = JsonRules('Achievements') //shares a universe of rules
// DEFINING RULES
// priority: integer, not exclusive; can be multiple 1's.  higher # = runs earlier
// inSeries(false): true|false.  Defaults to running in parallel.  Turn this on to run 'any' or 'all' in series
//   useful when needing to load data in the first few conditions
// action: callback to be executed when conditions passes
//         *** actions can modify facts
achievementRules.addRule({
  id: "age-range"
  priority: 1,
  conditions: {
    all: [{
     "id": "6ed20017-375f-40c9-a1d2-6d7e0f4733c5",
      // priority: integer, not exclusive; can be multiple 1's.  higher # = runs earlier
      fact: "eligibility",
      params: {
        equationId: '11111111-2222-3333-4444-5678901234567',
        eligibilityId: '11111111-2222-3333-4444-123456789012',
        fieldid: 'age'
      },
     "operator": "lessThan",
     "value": 45
   },
   {
    "id": "0941d27f-8154-439a-a720-ed4b48c196eb",
     "fact": "age",
     "operator": "greaterThan",
     "value": 21
   }]
 },
 action: {
    type: "reward",
    params: {
      reward: {
        type: "currency",
        detail: {
          currency: "points"
          amount: 50
        }
      }
    }
 }
})

achievementRules.addRule({
  all: [{
   "fact": "tags",
   "operator": "in",
   "value": '11111111-2222-3333-4444-123456789012'
 }]
})

// ADDING FACTS
// facts are persisted across each run() of the rules
achievementRules.addFact("eligibility", { cache: true }, (params, r, done) => {
  // api call to load demographic data
  // need access to facts
  let options = Params(params).only('eligibilityId')
  let eligibilityData = !r.facts.getFact('eligibility', options)
  if(eligibilityData) {
    eligibilityData = await request(`/users/${facts('userId')}/eligibilities/${params.eligibilityId}/data`)
    r.addFact('eligibility', { cacheKey: r.generateCacheKey('eligibility', options)}, eligibilityData)
  }
  done(null, eligibilityData.age)
}, { cache: true })

//    addFact OPTIONS
//      cache:  persists across each run of the rules engine, default: true
//              can also receive a callback, which can return true or false to denote caching
//      cacheKey: sets the cacheKey
//      generateCacheKey(fact, params): returns generated cache key based on inputs


// RETRIEVING FACTS
r.getFact(factName, params = {}) //searches the internal facts hash map


// RUNNING RULES
// rules.run(facts)
// all other methods should raise an exception if called after run()
achievementRules.run({
  userId: '0e03b9bd-36f9-4933-9ae8-2165baeaccde',
  performedAt: '2015-05-25',
  tags: ['11111111-2222-3333-4444-123456789012']
})

achievementRules.on('error', (r, rule) {
  // log failure reason
})
achievementRules.on('failure', (r, rule) {
  // log failure reason
})
achievementRules.on('action', (r, action) {
  // log failure reason
})

// HANDLING ACTIONS
// disadvantage is that if the persisted version changes, the code breaks
achievementRules.on('reward', (r, data) => {
  data
  // can this stop the rules?
  // alternatively, could just add some control properties to action
  //  e.g. { emit: reward, stop: true, data: {} }
})

achievementRules.onAction((r, rule, done) {
  let action = rule.action
  switch (action.type) {
    case 'reward':
      let grantAmount = action.data.reward.amount
      if(r.getFact('pointCaps')[action.data.currency] < grantAmount) {
        grantAmount = r.getFacts('pointCaps')[action.data.currency]
      }
      if(grantAmount > 0) {
        grant(grantAmount)
      }
      break;
    case 'pointCapReached':
      // is this the lowest point cap for this currency?
      let newPointCap = Math.max(r.facts.pointBalance - action.data.pointCap)
      r.facts.pointCaps = r.facts.pointCaps || {}
      if(!r.facts.pointCaps[action.data.currency] || r.facts.pointCaps[action.data.currency] < newPointCap) {
        r.facts.pointCaps[action.data.currency] = newPointCap
      }
      break;
    default:
      done()
      break;
  }
})

// when a rule runs, it works from the outside-in
// when it encounters a fact that is not defined, it will emit an error event, and continue to
// the next rule
//
//
// POINT CAPS
//  single user who meets certain criteria, limit to X points
achievementRules.addRule({
  id: "point-cap"
  priority: 100,
  conditions: {
    all: [{
     "fact": "age",
     "operator": "lessThan",
     "value": 45
   },
   {
     "fact": "pointBalance",
     "operator": "greaterThanInclusive",
     "value": 1000
   }]
 },
 action: {
    type: "pointCapReached",
    params: {
      currency: 'points',
      pointCap: 1000
    }
 }
})

//  family user, X points per, Y points per family
achievementRules.addRule({
  id: "point-cap"
  priority: 100,
  conditions: {
    all: [
      {
       "fact": "inFamily",
       "operator": "equal",
       "value": true
     },
     {
       "fact": "familyPointBalance",
       "operator": "greaterThanInclusive",
       "value": 5000
     }
    ]
 },
 action: {
    type: "pointCapReached",
    params: {
      currency: 'points',
      pointCap: 5000
    }
 }
})

// rate limits
achievementRules.addRule({
  id: "point-cap"
  priority: 100,
  conditions: {
    all: [{
     "fact": {
        name: "rateLimit",
        params: {
          per: 30
        }
      }
     "operator": "lessThanInclusive",
     "value": 1
   }]
 },
 action: {}
})

//  date ranges
achievementRules.addRule({
  id: "point-cap"
  priority: 100,
  conditions: {
    all: [
    {
     "fact": {
        name: "dateRange",
        params: {
          startAt: '1994-11-05T13:15:30.000Z',
          endAt: '2014-11-05T13:15:30.000Z'
        }
     },
     "operator": "equal",
     "value": true
   }
   ]
 },
 action: {}
})

// ORs: Total Cholesterol: ≤199 OR TC:HDL Ratio: ≤ 4.0 OR individual LDL, HDL, and triglycerides
achievementRules.addRule({
  id: "age-range",
  priority: 1,
  conditions: {
    any: [{
     "id": "6ed20017-375f-40c9-a1d2-6d7e0f4733c5",
      fact: "biometrics",
      params: {
        sponsorTagId: '11111111-2222-3333-4444-5678901234567',
        fieldId: 'LDL'
      },
     "operator": "lessThanInclusive",
     "value": 199
   },
   {
     "id": "6ed20017-375f-40c9-a1d2-6d7e0f4733c5",
      fact: "biometrics",
      params: {
        sponsorTagId: '11111111-2222-3333-4444-5678901234567',
        fieldId: 'TC-HDL-Ratio'
      },
     "operator": "lessThanInclusive",
     "value": 4
   }]
 },
 action: {}
})


// Group Rewards:
//    Offer incentive when more than 50% of the group completes a step program
//    Award bonus rewards to all participants that have completed the activity once the threshold is reached
// group.4184a91a-582e-4e9a-93da-8e34d5b458b3.program.abdcfe9f-5d6a-43af-8cb5-95b7f69cef44.completion-progress
achievementRules.addRule({
  id: "age-range",
  priority: 1,
  conditions: {
    all: [{
     "id": "6ed20017-375f-40c9-a1d2-6d7e0f4733c5",
      fact: "team_participation",
     "operator": "greaterThanInclusive",
     "value": 50
   },
   {
     "id": "6ed20017-375f-40c9-a1d2-6d7e0f4733c5",
      "fact": {
        name: "rateLimit",
        params: {
          per: 1
        }
      }
     "operator": "lessThan",
     "value": 1
   }]
 },
 action: {
    type: "groupReward",
    params: {
      reward: {
        type: "currency",
        detail: {
          currency: "points"
          amount: 50
        }
      }
    }
 }
})
achievementRules.run({groupId: '4184a91a-582e-4e9a-93da-8e34d5b458b3'})

// TODOs:
// cache; hash the fact to cache: https://github.com/puleos/object-hash
// input validatons; guard against bad input data
// README
// emit 'failure' when a rule doesn't pass
// recursive processing of any/all
// ----------
// controls: engine.stop(), engine.restart()
// support callback based facts and engine.run()
// might be nice to emit(action.type)
