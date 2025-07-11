I wanna make a graph describe cache control in my project
 
Data storages:
 
DB
 
ComfyAPI server in mem cache

(possibility) reverse-proxy cache, (only caches public data, e.g. /nodes/), (e.g. "Server: Google Frontend" in the Header for our case, or maybe nginx/caddy in general)
 
ReactQuery cache local-storage (shared by all pages in browser profile, instantly sync to other pages, caches all data for a short time, with staleTime=0 )
 
React-query in-mem cache (shared in a session, avaliable between user click to new page or go back in same page, caches all data)
 
 ----


and some senarios:

1. Claiming Node userflow

when change publisherId: from @Unclaimed into @publiser
User expect to claim a [publiserh=@unclaimed] node => to [publisher=@user]

[user] opens claim-my-node page
[browser] loads [@unclaimed] data
[User] saw [@unclaimed] node, and click [Claim] button
[Browser] calls [Claim] api
[comfy-api-server] db updated => invalidateMemCache of /nodes/[nodeId]
[browser] Invalidate all related nodes => /nodes/[nodeId] and /nodes
[browser] refetch with /nodes/[nodeId] NO-CACHE-HEADER to make all reverse-proxies invalidate their cache
[reverse-proxy] fetch server with that header
[comfy-api-server] response with latest data to reverse-proxy
[reverse-proxy] response with latest data to browser
[user] saw latest data [@user]!

2. Claiming Node userflow (with other user also watching)

[user2] doing other stuffs and seeing the [@unclaimed] node

[user] opens claim-my-node page
[browser] loads [@unclaimed] data
[User] saw [@unclaimed] node, and click [Claim] button
[Browser] calls [Claim] api
[comfy-api] db updated => invalidateMemCache of /nodes/[nodeId]
[browser] Invalidate all related nodes => /nodes/[nodeId] and /nodes
[browser] refetch with /nodes/[nodeId] NO-CACHE-HEADER to make all reverse-proxies invalidate their cache.
[reverse-proxy] fetch server with that header
[comfy-api-server] response with latest data to reverse-proxy
[reverse-proxy] response with latest data to browser
[user] saw latest data [@user]!

[user2] (re-focus to page), see [@unclaimed] data from browser cache for a moment (<100ms)
[user2-browser] because of re-focus, react-query just revalidate /nodes/[nodeId] without cache-control header), fetching from reverse-proxy without cache-control
[reverse-proxy] response cached data (already refreshed)
[user2-browser] shows refreshed data
[user2] saw latest data [@user]!
