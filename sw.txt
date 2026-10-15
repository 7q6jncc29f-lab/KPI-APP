/* KPI offline support: cache only public app code, NEVER user KPI values.
   Each device retains its own weekly figures separately in localStorage. */
const CACHE_VERSION='kpi-offline-v4-20261015';
const APP_URL=new URL('./index.html',self.registration.scope).href;
const CDN_URL='https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js';
const isOwn=(u)=>u.origin===self.location.origin && u.pathname.startsWith(new URL(self.registration.scope).pathname);

async function cacheExportLibrary(){
  try{
    const cache=await caches.open(CACHE_VERSION);
    const req=new Request(CDN_URL,{mode:'no-cors',cache:'reload'});
    const response=await fetch(req);
    if(response && (response.ok || response.type==='opaque'))await cache.put(req,response);
  }catch(err){console.warn('Export library offline cache will retry when online',err)}
}

self.addEventListener('install',event=>{
  event.waitUntil((async()=>{
    const cache=await caches.open(CACHE_VERSION);
    // The current published HTML is the fallback when the network disappears.
    await cache.add(new Request(APP_URL,{cache:'reload'}));
    await cacheExportLibrary();
    await self.skipWaiting();
  })());
});
self.addEventListener('activate',event=>{
  event.waitUntil((async()=>{
    for(const key of await caches.keys()){
      if(key.startsWith('kpi-offline-')&&key!==CACHE_VERSION)await caches.delete(key);
    }
    await self.clients.claim();
  })());
});
self.addEventListener('fetch',event=>{
  const request=event.request;
  if(request.method!=='GET')return;
  const url=new URL(request.url);
  if(url.href===CDN_URL){
    event.respondWith((async()=>{
      const cache=await caches.open(CACHE_VERSION);
      const cached=await cache.match(request);
      if(cached)return cached;
      try{
        const response=await fetch(request);
        if(response && (response.ok || response.type==='opaque')){
          await cache.put(request,response.clone());
        }
        return response;
      }catch(err){
        const fallback=await cache.match(CDN_URL);
        if(fallback)return fallback;
        throw err;
      }
    })());
    return;
  }
  // Network-first for HTML: newly uploaded releases can be fetched promptly,
  // while a force-close and reopen without a connection uses cached HTML.
  if(isOwn(url) && (request.mode==='navigate'||url.pathname.endsWith('/index.html'))){
    event.respondWith((async()=>{
      const cache=await caches.open(CACHE_VERSION);
      try{
        const response=await fetch(request);
        if(response.ok){await cache.put(APP_URL,response.clone());return response}
        const saved=await cache.match(APP_URL);
        return saved||response;
      }catch(err){
        const saved=await cache.match(APP_URL);
        if(saved)return saved;
        throw err;
      }
    })());
  }
});
