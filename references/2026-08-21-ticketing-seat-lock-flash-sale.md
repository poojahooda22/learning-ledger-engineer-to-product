# References: ticketing seat lock and flash-sale architecture (2026-08-21)

Keeper links for the Zomato / District seat-lock teardown.

## The real scale event (Coldplay India onsale)
- Pollstar, "BookMyShow Buckles Under Coldplay Onsale In India As 13 Million Try To Secure Tickets" (Sept 2024). Confirms 13 million concurrent, the crash, the scale. https://news.pollstar.com/2024/09/26/bookmyshow-buckles-under-coldplay-onsale-in-india-as-13-million-try-to-secure-tickets/
- Malay Mail, "Coldplay's three concerts in India sell out in 30 minutes" (Sept 2024). Sellout time, 178k-scale demand. https://www.malaymail.com/news/showbiz/2024/09/29/coldplays-three-concerts-in-india-sells-out-in-30-minutes-tickets-resold-for-as-high-as-rm49261/151988
- BookMyShow.Live on X, the added India show, waiting room timing and the move to randomized queue position. https://x.com/Bookmyshow_live/status/1857681822085427257

## Zomato / District ticketing business
- TechCrunch, "Zomato buys Paytm's entertainment ticketing business for $244 million" (Aug 2024). Paytm Insider + TicketNew into District. https://techcrunch.com/2024/08/21/zomato-buys-paytms-entertainment-ticket-business-for-244-million
- District by Zomato (movies, events, IPL, concerts). https://www.district.in/

## The seat-lock and double-booking pattern
- Redis SET command docs. Atomic NX + PX in one command; why the old SETNX then EXPIRE pattern is deprecated. https://redis.io/docs/latest/commands/set/
- OneUptime, "How to Implement a Booking Lock System with Redis." SET NX PX, TTL auto-release of abandoned holds. https://oneuptime.com/blog/post/2026-03-31-redis-booking-lock-system/view
- DevelopersVoice, "BookMyShow Seat Selection Architecture: Distributed Locks, Payment Sagas and Zero Double-Booking at Scale." https://developersvoice.com/blog/practical-design/scalable-net-ticketing-architecture-distributed-locks/
- techinterview.org, "System Design: Ticketing System (Ticketmaster)." Seat states, holds, waiting room, viewing-vs-booking traffic ratio. https://www.techinterview.org/post/3233463451/system-design-ticketing-system-ticketmaster/

## Virtual waiting room (admission control) pattern
- Queue-it, "How Queue-it Works." Edge connector, HTTP 302 to the waiting page, token verification, FIFO plus randomization, configurable drain rate. https://www.queue-it.com/developers/how-queue-it-works/
- AWS APN blog, "How to manage peak traffic on AWS using Queue-it's Virtual Waiting Room." DynamoDB queue state at hundreds of thousands of transactions per second, CloudFront edge checks. https://aws.amazon.com/blogs/apn/how-to-manage-peak-traffic-on-aws-using-queue-its-virtual-waiting-room
