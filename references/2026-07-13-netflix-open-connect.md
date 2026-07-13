# References: Netflix Open Connect (2026-07-13 teardown)

Keeper links on Netflix's private CDN, proactive fill, and play-time steering.

## Netflix primary sources

- Netflix Technology Blog, "Netflix and Fill" (2016). Proactive fill, peer fill
  and tier fill, fill clusters, fill-master cascade, off-peak fill windows.
  https://netflixtechblog.com/netflix-and-fill-c43a32b490c0
- Netflix Technology Blog, "Content Popularity for Open Connect." Per-file
  popularity ranking vs per-title, regional prediction, cache efficiency gains.
  https://netflixtechblog.com/content-popularity-for-open-connect-b86d56f613b
- Netflix Technology Blog, "Serving 100 Gbps from an Open Connect Appliance."
  FreeBSD, NGINX, sendfile and async sendfile, kernel TLS (kTLS), NUMA pinning;
  ~90 Gbps of 100% TLS traffic on the default stack, pushing past 100 Gbps.
  https://netflixtechblog.com/serving-100-gbps-from-an-open-connect-appliance-cdb51dda3b99
- Netflix Technology Blog, "Driving Content Delivery Efficiency Through
  Classifying Cache Misses." How Netflix studies and reduces the sub-5% miss rate.
  https://netflixtechblog.com/driving-content-delivery-efficiency-through-classifying-cache-misses-ffcf08026b6c
- Netflix Open Connect, "Open Connect Overview" (PDF). Control plane in AWS vs OCA
  data plane, steering flow, appliance families, deployment models.
  https://openconnect.netflix.com/Open-Connect-Overview.pdf
- Netflix Open Connect, appliance specifications page (storage vs flash OCAs,
  capacity and throughput).
  https://openconnect.netflix.com/en/appliances/

## Netflix Open Connect Partner Help Center

- "Fill patterns." Fill window timing, peer fill vs tier fill mechanics.
  https://openconnect.zendesk.com/hc/en-us/articles/360035618071-Fill-patterns
- "Network configuration." BGP session required per appliance to participate in
  steering and delivery.
  https://openconnect.zendesk.com/hc/en-us/articles/360035533071-Network-configuration
- "Requirements for deploying embedded appliances."
  https://openconnect.zendesk.com/hc/en-us/articles/360034538352-Requirements-for-deploying-embedded-appliances

## Third-party and community

- FreeBSD Foundation, "Case Study: Netflix Open Connect." Serving stack, kTLS
  upstreamed to FreeBSD, throughput.
  https://freebsdfoundation.org/wp-content/uploads/2021/03/Netflix-Open.pdf
- APNIC Blog, "Netflix content distribution through Open Connect" (2018).
  Proximity rank, BGP-based steering, embedded vs IX deployment.
  https://blog.apnic.net/2018/06/20/netflix-content-distribution-through-open-connect/
- Wikipedia, "Open Connect." Background, history, scale.
  https://en.wikipedia.org/wiki/Open_Connect

## The one-line takeaway

Stop moving the firehose at request time. Split control plane (AWS, thinking) from
data plane (OCAs, streaming), predict per-file popularity per region, and push the
bytes into servers physically inside the ISP overnight during an off-peak fill
window (cascaded peer -> tier so origin is not hammered). Then the live request is
matching (OCAs that have the file and are healthy) then ranking (BGP proximity),
returning a ranked URL list. Target: >95% of bytes served from a box a few km away.
