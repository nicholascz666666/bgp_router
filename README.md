# BGP Router

## Approach

In general, we followed the implementation order given in the project specs. 

#### Basic Support

1. After receiving an update, our router saves a copy of the new network to a list of received updates in case we need to retrieve this data later. 
2. We also append the update to a list of non-aggregated routes, in order for easier use in disaggregating. 
3. We then add an entry to our forwarding table and add the update to our routes. 
4. Finally, we attempt coalesce on the routes to see if there are any routes that can be aggregated. Once this is all done, we forward the update on to peers. 

#### Dump/Table Messages

1. We wrote a simple loop to go through each route in the routing table, adding each route's network, netmask, and peers to a list. 
2. A message was then sent through the package containing this table with the type label "table". 

#### Selecting the Best Path

1. Routes are sorted based on localpref
2. If the localprefs of the first two routes (given there are more than two routes) are equal, all routes with matching
   localpref are then sorted by self origin
3. If the self origin of the first two routes are equal, all routes with matching self origin are then sorted by ASPath
   length
4. If the ASPath length of the first two routes are equal, all routes with matching ASPath lengths are then sorted by
   origin
5. If the origins of the first two routes are equal, all routes with matching origins are then sorted by IP

A helper function was written to prune the routes to only those with attributes matching the first in the list to ensure
that priority was still given in the order presented above.

#### Revoke/No-route Messages

1. We save a copy of the revoke in case we need it later. 
2. We remove the entry from our list of updates.
3. We then disaggregate our routes
4. Finally, we send an update to all peers

#### Restricting Provider/Customer Relationships

1. We loop through the routes we have saved and remove the route if they are any of the following:
   - Peer to peer
   - Peer to prov
   - Prov to peer

#### Longest Prefix Matching

We loop through the routes and check if the route's prefix is greater than the current maximum. If it is, we replace the current maximum with it. If it is equal, we append the route to the list. 

#### Path Aggregation and Disaggregation

Coalescing the paths was rather difficult. The program was written as follows:

1. A route set was created and a counter for how many times a route was added.
2. Each route was first checked again every route already in the set to see if they could be aggregated; if no routes
   were in the set yet or if the route could not be aggregated with any, it was added to the set.
3. If a route could be aggregated, the program recalculated the aggregated network and netmask. This route would then
   replace the route it was aggregated with.
4. Once all routes were accounted for, the program would run recursively until each route was added to the set without
   needing to be aggregated with another route. This ensured that if multiple routes that could be aggregated existed,
   all of the routes would be aggregated.
5. Finally, this function was run after every update message.

We saved a copy of non-aggregated paths, which we continuously update when revoke messages are sent in. When a disaggregation call is made, we revert back to this list of saved paths. 

## Challenges

- When performing path aggregation, it was difficult to handle cases where there were more than two paths that could be
  aggregated
- Path aggregation also presented challenges when it came to structuring the code, due to the many possible cases
- Figuring out an efficient way to disaggregate paths was also challenging. In the end, we decided on saving a separate list of disaggregated paths

## Testing

We tested our code by running the sim on the Khoury Linux machines. While we broke the project up into parts to
complete, our final testing involved running all simulation tests consecutively to ensure that all tests would pass. 
