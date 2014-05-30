# Autoscale - Triggers

## Prerequisites

You will have run through the last article and have a bash command line open with certain variables set up:

 * token = Your API token
 * auth = The string: "X-Auth-Token:$token"
 * group_id = The ID of the scaling group that you just made
 
## Scaling Policies

There are two types of scaling policies; web-hooks and scheduling. You can use the web interface to set up scheduling policies, so we'll focus on web-hooks here.

The big advantage of a web-hook trigger is, we can configure the monitoring url to hit it.

When the policy is hit, it can grow or shrink the number of web nodes by a certain number, a percentage, or just set them to a target amount.

## Adding the policies

We'll add two web hook policies, one to grow the number of web-heads by one, and one to reduce them.

Lets just [list the current policies](http://docs.rackspace.com/cas/api/v1.0/autoscale-devguide/content/GET_getPolicies_v1.0__tenantId__groups__groupId__policies_autoscale-policies.html) to make sure we're starting with a clean state:

    http $url/groups/$group_id/policies $auth

There's [only a few parameters](http://docs.rackspace.com/cas/api/v1.0/autoscale-devguide/content/POST_createPolicies_v1.0__tenantId__groups__groupId__policies_autoscale-policies.html#POST_createPolicies_v1.0__tenantId__groups__groupId__policies_autoscale-policies-Request) that we need to set:

 * name - "Add a server"
 * cooldown - 600 (6 minutes between calls, after adding one, give it a chance to start taking load before adding more)
 * change - 1 (just add one server)
 * type - webhook

Lets add this policy:

    policy=$(echo '[{"name":"Add a server","cooldown":600,"change":1,"type":"webhook"}]' | http POST $url/groups/$group_id/policies $auth)

You can check out the results with:

    echo $policy | jq .

Lets store the policy id for future calls:

    policy_id=$(echo $policy | jq '.policies | .[] | select(.name == "Add a server") | .id' | sed s/\"//g)
    policy_url=$url/groups/$group_id/policies/$policy_id

## Adding the webhook

Now that we have the policy we need to add a [webhook](http://docs.rackspace.com/cas/api/v1.0/autoscale-devguide/content/autoscale-webhooks.html) to it. First lets just list the webhooks that already exist (it should be empty):

    http $policy_url/webhooks $auth

Now we'll add one. Nice and easy:

    webhook=$(echo '[{"name":"Add one - cpu"}]' | http POST $policy_url/webhooks $auth)

Lets have a look at the nice webhook list and urls it has give us now:

    echo $webhook | jq .

We'll store the url for later:

    trigger_1=$(echo $webhook | jq '.webhooks | .[] | select(.name == "Add one - cpu") | .links | .[] | select(.rel == "capability") | .href' | sed s/\"//g)
    
We can now add a server to our build out just by doing a POST to that url:

    echo $trigger_1
    http POST $trigger_1

## How do we go down again ?

To go down again, we just make a new policy for removing one web head. We just change the name and quantity to -1:

    policy=$(echo '[{"name":"Remove a server","cooldown":600,"change":-1,"type":"webhook"}]' | http POST $url/groups/$group_id/policies $auth)
    policy_id=$(echo $policy | jq '.policies | .[] | select(.name == "Remove a server") | .id' | sed s/\"//g)
    policy_url=$url/groups/$group_id/policies/$policy_id

Add the webhook for it:

    webhook=$(echo '[{"name":"Remove one - cpu"}]' | http POST $policy_url/webhooks $auth)
    trigger_2=$(echo $webhook | jq '.webhooks | .[] | select(.name == "Remove one - cpu") | .links | .[] | select(.rel == "capability") | .href' | sed s/\"//g)

Now we can take away one server from our list:

    http POST $trigger_2

## Conclusion

Now we can make policies and web hooks .. in the next article we'll look at tying that in to the monitoring system, having it trigger a web hook to add a server when the CPU on Web01 goes over 90% for x seconds.  Later we'll look at a custom monitoring alert, that'll scale up if any web head hits the Apache Max Clients setting, and scale down if any web head gets below 3 concurrent requests.
