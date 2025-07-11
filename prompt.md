I wanna make a graph describe cache control in my project
 
Data storages explain:
name | how it works| persist time | shared by | how to update/invalidate
[DB] DB, can be treated as only one, persist forever, update by comfy-api server.
[server-mem-cache] ComfyAPI server in-mem cache, may have multiple instances, shared by single instance, clean when new commit deployed, can be invalidate by call invalidateCache which is always called when data updated
[reverse-proxies-cache] (possibility) reverse-proxies cache, (only caches public data, e.g. /nodes/), (e.g. "Server: Google Frontend" in the Header for our case, or maybe nginx/caddy in general, or ISP deployed cache which is not controlled by us), usually in different server between users in different area, shared by multiple user, persist: cache-control header allowed or may shorter, can only possible to update/invalidate by client refetch with "cache-control: no-cache" header if not expired.
[browser-disk-cache] fetch() similar to reverse-proxies cache, stored in disk, can be clear in browser's site setting
[localStorage] ReactQuery cache local-storage (shared by all pages in browser profile, instantly sync to other pages, caches almost all data for a short time, with staleTime=0 so it always refetch ), persist across user's system reboot.
[sessionStorage] React-query sessionStorage cache (shared in a session, avaliable between user click to new page or go back in same page, caches all data), registry-web currently don't use this yet
[queryClient] React-query in-mem cache (Avaliable between user click to new route or go back in same route, until user refresh the page)

----

invalidation methods:

1. browser:: react-query invalidateQueries: update localStorage and sessionStorage cache, but not proxies-cache and mem-cache and [browser-disk-cache]
2. browser:: refetch with NO-CACHE-HEADER (will ensure proxies-cache and browser-disk-cache updated, but not mem-cache yet, and only updates the user's ISP deployed Cache, not for other user's ISP)
3. server:: ComfyAPI invalidateCahce method (will be called after when ever data update success): updates mem-cache, but not proxies-cache and mem-cache

----


and some senarios:

1. our current Claiming Node userflow

when change publisherId: from @Unclaimed into @publiser
User expect to claim a [publiserh=@unclaimed] node => to [publisher=@alice]

[alice] opens claim-my-node page, loads [@unclaimed] data
[alice] saw [@unclaimed] node, and click [Claim] button
[alice] calls [Claim] api, and Invalidate all related nodes => /nodes/[nodeId] and /nodes
[one-of-server] => update [db] => invalidateMemCache of /nodes/[nodeId] => response 'claimed'
[alice] show button: goto claimed node page /publisher/[@unclaimed]/nodes/[@node]
[alice] *click
[alice] in /publisher/[@unclaimed]/nodes/[@node], show [@unclaimed] for a short time, and revalidating while stale (and seeing stale data about 200ms)

[alice] and got latest [@alice] this node have been updated, redirect to /publisher/[@]/nodes/[@node]
[proxies] fetch upstream without no-cache header
[one-of-server] response with cached data to reverse-proxy
[proxies] response with latest data to browser
[alice] (shows claim success, and navigate to page)
[alice] saw latest data [@alice]! 


1. Claiming Node userflow

when change publisherId: from @Unclaimed into @alice
User expect to claim a [publiserh=@unclaimed] node => to [publisher=@alice]

[alice] opens claim-my-node page, loads [@unclaimed] data
[alice] saw [@unclaimed] node, and click [Claim] button
[alice] calls [Claim] api, and Invalidate all related nodes => /nodes/[nodeId] and /nodes
[one-of-server] => update [db] => invalidateMemCache of /nodes/[nodeId] => response 'claimed'
[alice] refetch /nodes/[nodeId] with NO-CACHE-HEADER to make all reverse-proxies invalidate their cache.
[proxies] fetch upstream until server with NO-CACHE-HEADER header
[one-of-server] response with latest data to reverse-proxy
[proxies] response with latest data to browser
[alice] (shows claim success, and navigate to page)
[alice] saw latest data [@alice]! 

2. Claiming Node userflow (with other user also watching)

[bob] doing other stuffs and seeing the [@unclaimed] node, same ISP with alice
[carol] doing other stuffs and seeing the [@unclaimed] node, different ISP with alice and bob

[alice] opens claim-my-node page, shows [@unclaimed] data
[alice] saw [@unclaimed] node, and click [Claim] button
[alice] calls [Claim] api
[one-of-server] => update [db] => invalidateMemCache of /nodes/[nodeId] => response 'claimed'
[alice] Invalidate all related nodes => /nodes/[nodeId] and /nodes
[alice] refetch with /nodes/[nodeId] NO-CACHE-HEADER to make all reverse-proxies invalidate their cache.
[proxies-ISP] fetch upstreams with NO-CACHE-HEADER, got latest [@alice] data (and set cache to [@alice])
[proxies-GCP] fetch upstreams with NO-CACHE-HEADER, got latest [@alice] data (and set cache to [@alice])
[one-of-server] response with latest data to downstream
[alice] saw latest data [@alice]!

[bob] (re-focus to [@unclaimed] page), see [@unclaimed] data from browser cache for a moment while revalidating, (<100ms)
[bob] because of re-focus, react-query just revalidate /nodes/[nodeId] without cache-control header, fetching from reverse-proxy without cache-control
[proxies-ISP] response cached @ data (because refreshed by alice's fetch)
[bob] shows refreshed data
[bob] saw latest data [@alice]!

[carol] (re-focus to [@unclaimed] page), see [@unclaimed] data from browser cache for a moment while revalidating, (<100ms)
[carol] because of re-focus, react-query just revalidate /nodes/[nodeId] without cache-control header, fetching from reverse-proxy without cache-control
[proxies] response cached data
[carol] shows refreshed data
[carol] saw latest data [@alice]!

And explain we only need refetch no-cache-header once from the user who's Claiming the node, and all reverse-proxies()'s cache will be updated!

Q: What if alice claimed the node and crashed before refetch /nodes/[nodeId]?
A: the user will try claim it again and got error "already claimed", then browser will fetch the node with NO-CACHE-HEADER again.



## Task:

1. first, give me a table that explains Data storages

name | device | how it works| persist time| how to update/invalidate

2. add invalidation methods table

per stoages as cols, start from queryClient, end with db
per invalidation methods as rows

checkmark if updatable, add * for partial update (e.g updates is only effect to one user or a small part of users)
starmark as usually already latest.
leave empty when no changes


2. draw graphs for scenarios, using UML-like graph to describe cache update relation

ensure browser (Alice|bob) ,localStorage, proxies, ComfyAPI Server, Database is draw in the graph

feelfree to use any pkg to draw the graph, render into index.html, use bun install if necessary
index.html will be served by vite (I will run by my self)