# FreightCartMod 
Fork of Steveman0's Freight Cart Mod for FortressCraft Evolved

##TODO:
1. encapsulate massstorage and hopper logic in an interface, so nobody has to care which is which in the rest of the code.

2. Fix serialization. Try real hard, this should be cast in stone as much as possible.
 * Figure out what fields need to be added to mob, FCM, station, etc. and add them.
 * Update serialization
3. Add IPM calculation to the stations. 
3. Add the route creation logic to FCM. Simple 2-stop routes only at first.
3. Update cart logic
3. Add route & cart auto-naming
4. Add cart de-spawning, and re-spawning with specific routes pre-loaded
5. Update systemmonitor screen. All the data structures should be in place, so we can make it fairly complete even at this point. Warnings and all! (I wonder if we can do colored buttons? Probably not...) Add control over individual carts through the systemmonitor screen too.
6. Add more complicated route combining and such.
7. Update cart logic to include going between similar stations
8. Remove vanilla minecarts and scrap track, and put our minecarts and logic stuff behind a research barrier
9. Add enhancements interface to station screen, and add enhancment items with crafting recipies etc. 


##The Plan (TM)
Freight offers/requests by catagory. Or substring. Or something.
Freight offers/requests with priority (so we can have a mass storage that holds bars until they're needed)
New Scheduling Algorythm

Thoughts on IPM: From the perspective of the storage unit, there's different phases that a manufaturing plant goes through:
Empty: No bars/items are being processed or stored anywhere in the system. Storage is empty, IPM is zero.
Full: Every conveyor and hopper in the line is filled. plants have a crafted item, but no where to put it. Zero IPM.
Steady state: the bottle necked somewhere, constant IPM, less than surge.
Surging: Transitioning from empty to full. storage was empty, IPM temporarily at elevated levels.

Graphing IPM in game would be fun :) Have to just save the interesting parts of the chart though... a flat chart saying "we've been at zero for the past five hours" isn't very interesting.

But does any of this matter if we just use a weighted floating average? I mean, we can remove the "false zeros" like empty storage, but beyond that, how much fiddling with the signal is worth it?

##Design goals for the new scheduling algorythm.

* I want this to be based (even if only behind the scenes) on IPM, not "amount" per se. Definately set the amount, but the system needs to dispatch MANY carts in some circumstances, even if the raw amount figures don't justify it.
* I want carts to essentially stake out a "route" between a producer and a consumer.
* I want the player to be able to temporarily kick the priority of a particular item (I'm waiting on this one...) (reset after five minutes?)
* I want carts to automatically take secondary loads going the other direction if they can (same cart)
* I want them to keep "empty this" hoppers empty (or at least with some empty space in them), and "fill this" hoppers full.
* I want to be able to set up a smelter with a single hopper and station, set to "offer bars, accept ore" and not worry about type. Simular with crystal clocks etc.
* I want to be able to set up ANY factory with a single hopper and station, set to "49% iron bars, 49% nickel bars, offer steel bars" and the like.
* I want the above setup to communicate with the manufacturing plant to get desired items, and offer unneeded (maybe we changed the recipe?) and recipe results.
* vanilla minecarts should be entirely hidden: use ours instead.
* Stations should be upgradable with add-ons, just like the ore extractors and LPT's are.
* Mass storage interface should require an add-on, and speed of interface should also be a limit. (Maybe we can use the vanilla mass storage flyer things? That'd be cool! Launch multiple of them from one station. Maybe require power?)
* vanilla minecart logic should still be available with our stations. Just hide the UI till they've unlocked it.
* It should have perhaps two levels of "advanced minecart logistics" with all these upgrades. Maybe more as more advanced features are implemented?
* Consider making some upgrades FF only.
* Idle carts MUST be clearly marked as idle. Preferably by going to a local parking area. Depots?
* The central system controler might be unique to the track system.
* Minecarts get charged at Depots and take power to travel? Have to design it so there is NO possibility of getting "stuck" *unless* the player changes the track system.
* Minecarts display their contents, type, and charge level on their side.


#Design
IPM transported by a particular cart is a function of load/unload time, transit distance, and cart capacity. Based on round trip. Remember to figure in secondary routes (i.e. returning empty fuel canisters). The first part is a faily simple function.  The second? More complicated, but maybe we can add a fudge factor - after all, the point of a secondary route is that it doesn't take much additional time.  


Every station needs to keep track of IPM, either produced, or consumed, depending on if the item is offered or requested.

Remember, the goal is to keep the hopper "mostly full," so there should be some slack for not getting it exactly right. Default to last ten minutes, maybe user adjustable? For fast-consuming lines feeding slow-consuming needs (plasma drill heads?) another hueristic may be needed.

##Station Behavior
* Stations should take enhancers. In particular, load/unload rate enhancers, make mass storage require a mass storage interface, and the speed depend on the enhancement loaded. Also consider "wide lip" or something to accomidate improved minecarts?
* IPM figures should be updated once per second.
* Maximum change for the "steady change threshold" should be about equal to the max that conveyors can remove.
* Change figures need to count items added/removed by the freight station as part of inventory/against inventory.
* IPM should probably be a weighted floating average.
* IPM is a float, and is per-item offered or requested.

###IPM produced:
* Timer stops when the storage is full.
* Steady increase counts as production.
* Sudden drops (player removes items) doesn't count against production.
* Sudden increase (player adds items) doesn't count towards production.
* Steady drops don't count against production?
* Anytime there's no change either way, stop the clock.

###IPM Consumed: Basically the opposite of IPM produced.
* Timer stops when storage is empty
* steady decrease (presumably conveyors) counts as consumption, but steady increase doesn't count against it.
* Sudden increases don't count against consumption, but subsequent steady drops do.
* Sudden decreases don't count towards consumption.
* If blocked, (zero removals) don't update IPM.

These rules for IPM should allow the player to "just request items needed" and get an efficient result. Note that requesting even one item could be enough - the should "overfill" to keep up with IPM consumed, as the IPM doesn't update when the hopper is empty. That, in turn, may result in a cart waiting at the station for more space to become available.

That's OK, if consumption is blocked, then the IPM drops till (current contents /IPM < round trip time) and the cart is rescheduled, dropping it's excess off in the excess storage bin.

Bad idea - because when consumption resumes again, we won't have valid IPM figures. So, instead of that, stop updating IPM if no removals. And set a "blocked" flag or timer instead.

###Dump station
We'll need a "dump station" that has a low-priority accept everything for dumping excess, to avoid deadlocking full carts with unwanted goods. Should also obviously have a high-priority offer all to prevent such backlog from building up.

Maybe we can auto-designate a dump station by zero IPM but has contents? 'cause player dumping into it doesn't cause IPM, and neither does a cart dropping things off...

###Intermediate storage area?
But what about intermediate storage areas?
What should be their production/consumption figures? Are they now obsolete? How do we deal with a hopper/mass storage that has two stations, one which takes and one which gives? What should it's theoretical IPM figures be? Invalid, probably. Infinite until empty/full?

##Cart behavior
* Should *totally* ignore stations not on their itineratry.
* If we don't have an route, head for the nearest depot for storage.
* If our station is blocked, check if the next station is near (<5 blocks) and providing/accepting same thing. Procede there.
* If our next station is accepting, and we don't have enough (and we have more space) (calculate ETA vs IPM, including carts scheduled to arrive before us)
	then check and see if our last station(s) provide, and maybe go pick up some from them.
* Same in reverse for leaving an accepting station while still having items in our hold.
* Should leave (go to nearest junction, or on to next stop?) and then come back if their material is "full to spec" or just "full" but with other stuff. This is to allow other carts (missing ingredients?) a chance to step in. This is assuming (contents/IPM > round trip time), of course.
* If the next consumer(s) are blocked, then don't pick anything up for them.
* If the next consumer(s) are blocked, and we already have a cargo for them, just go there and wait. If still blocked after 30s, visit nearest junction. If still blocked after doing that four times:
	* If the next producer's corresponding consumers(s) are blocked, skip them all. If that ends up as "don't do anything" then drop the route and head to the nearest depot.
* If the storage full but doesn't have anything we want, leave regardless.
* If the storage isn't full, but doesn't have anything we want, wait 30s, unless "wait for full" is set, in which case we keep waiting until we're full or the storage is.

##system monitor screen
* derailing risks - give local junction, x,y,z, or some other indication of location.
* derailed carts - same.
* Keep track of underruns on comsumers: suggest adding more storage. Note that hoppers only hold 100, suggest switching to mass storage for more accurate IPM figures, and to avoid emptying it when a demand spike hits.
* Do we have an excess supply drop location? request the user make one. Or request it be mass storage, if it's full.
* Lacking carts? Cite specific needs that aren't being adaquately met. Suggest bigger/faster carts if that'll make a difference.
* <100% scheduled IPM is a problem.
* Report for each station
	* Which carts are coming, and how far away they are, current contents, contents designated for this station, etc. (seldom used, so we can use the accurate square root using distance calc)
	* Scheduled IPM need met
	* Contents of currently attached storage
* Report, for each cart, current location, current contents, route with quantities, ETA's, efficiency percentages, etc.
* Sort (and sub-tree) cart report by item, distance, station, ???
* Stations should also be sorted (grouped, really) by items offered, distance, name, etc.
* Auto-name stations based on needs/offers
* Auto-name carts based on route and number sharing that route.
* List juntions, in red if they have problems. Group stations under them.
* Report under-supply of anything.

##Scheduling Algorythm (The real meat of the matter)
* Must keep a list of all stations, carts, routes, and junctions on this track network
* Carts must point at routes, AND routes point at carts.
* Algo is kicked off based entirely off IPM needs.
	* Zero IPM and empty storage gets temp. route only.
* Limits load on providers based on IPM, or stock IFF it's dump station.
* Seeks to combine routes occasionally
* Searches amoung existing carts for a "deadhead" section that could be repurposed.
* Assigns both temporary (IFF from dump station) and repeating routes.
* Prefers scheduling more advanced carts to lesser ones. Should be a seamless transition to higher-tier carts and retiring lower-tier ones.
* Leaves lower-tier carts only if they're the only cart on a fully satisfied route.

###Scenarios:
	Cart goes back and forth to a remote turbine, taking fuel canisters and returning the empties.
	Cart goes out to a remote manufacturing site, taking sufficient quantity of several different needs and bringing back several different items.
	Cart goes around a loop, picking up and dropping off items several times along the way.

Method for evaluating different scheduling option: 
	Calculate efficiency: Percent of travel time carts run percent of empty.
	Calculate satisfaction percentage: Percent of time consumers are estimated to be empty
		Applies IFF there aren't enough carts. Probably give same sorting order as above anyway.

###Need sorting:
	New stations get only enough carts assigned to fill requested inventory amount. (and not more than storage capacity) until they have a steady IPM (<?% change in ? seconds?)
	Older stations should have 100+ percentage IPM filled, essentially being topped off to 100% filled every time a cart comes by. Note that the carts may not be entirely filled.


###Actual Algo:
1. Every second, check all stations for unmet IPM needs.
1. If a station vanishes, remove it *and the associated pickup/dropoff* from all routes, adjusting IPM satisfaction percentages accordingly.
1. If any storage is empty (or significantly below target) schedule a temporary route.
	* significantly below empty: We can get a cart there to refill without displacing a permanently routed carts inventory which is already scheduled to arrive. Or will be empty before the estimated arrival of next delivery. Based on IPM and ETA's.
	* May want to reseve a few fast carts for this purpose.
	* Need to search available carts (don't use scheduled carts for temp routs)
	* Find nearest producer who has inventory and IPM available.
	* No scheduled cart? done.
	* Find ETA from new cart to producer to consumer.
	* Does new cart arrive before? How much inventory can we safetly fill?
	* If greater than 10 or so, do it.
1. If any IPM need is < 100% filled schedule a permenant route
	Only permenant routes fill IPM needs.
	 (Carts auto-space? May do that automatically based on "waiting for dropoff" times.)
	Route is a seperate first-class object, that perists even if there are no carts on it.
	Routes exist even without carts, because we may temporarily de-assign carts if the consumer is blocked.
	* Is there an existing route that fills this need that has a producer that isn't entirely taken up? Just add a cart.
		* This addresses the case where the consumer was blocked, casing all carts to head to the depot.
	* Otherwise, we need a new route:
		* Make several candidate routes. For each one,
			* Calculate Utilization precentage and round trip time
			* Calculate idle time/number of carts required to fulfill all consumers
			* Calculate Utilization percentage impact on old route: +/-
			* Freight/ore/station limits validation (check all stations, make sure it's possible to assign carts to this route)
			* All carts must meet servicing requirements of all items/stations.
				* I.E. if the route says "pickup iron bars" Don't assign ore carts.
				* If the route calls for having two items in the hold, don't assign ore.
				* (And reject if the existing route already has ore carriers assigned to it.) 
			* Add to candidate routes
		* Construct simplest route:
			* Find a nearest producer who's IPM isn't taken up.
			* Add to route
			* Add a zero for impact on old.
			* Add consumer.
		* Find routes that cart the same thing that have idle time.
			(Maybe it can deliver half the load to the old one, and half to the new)
		* Is there a route satisfying this need that just needs another producer?
			* Add another producer to existing route. 
		* Find routes that can spare that much time from at least one cart. Are they close enough? Try to split off a cart onto a new combined route.
			* Carts*empty_time < B.time
			* layout proposed new route.
			* Add Permutations of route to the list (Expensive! I wonder what we can do to make this simpler. Look up "Vehicle Routing" problem.)
				* Eliminate premutations that require carrying capacity for the new route, but the old route has used it up.
			* Add to candidate routes
		* Sort candidate routes by utilization efficiency (times proposed number of carts on each route, of course.)
		* Select the top route, add any nearby producers/consumers of the same item.
		* Add carts (or request carts be added) until the consumer IPM is satisfied.
			* Find all carts that don't have a route, or have a route that reads "report to Depot"
			* Send requests to depot nearest producer to launch X carts to Y route.
		* Done
					
					
				 
		
		
		

1. If just spawned, and where given a need to fill by the station, do that.
1. Sort station list on "nearby" stations. Use distance estimate, not track distance (largest of (x,y,z) + 1/2 of the other two) (note that stations not on this track network are bottom of list)
1. Stop once ten or so stations with at least 10% unmet need are found?
1. Prioritized based on percentage IPM met by currently assigned carts, take the top onPrioritized based on percentage IPM met by currently assigned carts, take the top onee(unless launched for a specific need, then use that one)






