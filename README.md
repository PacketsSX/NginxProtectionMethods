**WORK IN PROGRESS, THIS IS THE BASE BEGINNING OF EXPLANATIONS AND DOCUMENTATION FOR ALL RATELIMITING AND PROTECTION METHODS.**

THESE ARE DIFFERENT CONFIGURATIONS FOCUSED ON PREVENTING THINGS SUCH AS DDOS ATTACKS, FOCUSING ON AS MUCH UPTIME AS POSSIBLE.


Nginx's ratelimiting system is known as "a leaky bucket", referring to a bucket full of water with a small hole, and depending on the parameters of that hole, allows X-Y amount of water to leak through. In simple terms, requests to the host arrives at a random and unfixed rate, but rate limiting makes sure that requests are still served at a fixed reasonable rate.

**WHAT ARE ZONES?**

In Nginx, a "zone" is a shared memory area that worker processes use for storing and sharing information. This is highly often used in rate limiting, as it is a feature that requires workers to read, write, and share data amongst themselves. Most if not all of these configurations will involve allocating and specifying the zones that clients will be assigned to, so make sure you have somewhat of an understanding of this feature.
Most of the time, these zones will keep and store the different states of each client, mainly information of how many requests this client has made, how large the size of the requests are and other information which is needed in order to know where and when and to who to enforce blocks and error messages.

Keep in mind that zones also keep track of active keys and the metadata of the cached files and resources. This allows workers to be more efficient, checking first if the request the client has made is already pre-cached or if it needs to request that page.


**LIMITING REQUESTS FROM A CLIENT PER SECOND**

    limit_req_zone $binary_remote_addr zone=afzone:10m rate=3r/s;
    
    server {

        location /index* { 
            limit_req zone=afzone;
        }

"limit_req_zone" is where the shared memory zone for workers is created. This zone is named "afzone" and has a size limit of 10 megabytes. The zone called "afzone" will only fullfill up to 3 requests per second from one single client.

After creating and defining the parameters of this zone, we enforce that zone on webpages and links involving /index, in this case both .php and .html.

If someone exceeds 3 requests per second, their excessive requests go into the zone, and the client's requests are then delayed. Once the zone becomes full, the client will receive a 503. 

KEEP IN MIND: ACCORDING TO NGINX; 

"The optimal size of the shared memory zone can be counted using the following data: the size of $binary_remote_addr value is 4 bytes for IPv4 addresses, stored state occupies 128 bytes on 64-bit platforms. Thus, state information for about 16,000 IP addresses occupies 1 megabyte of the zone."

**RATE LIMITING ALONE STOPS A GIANT CHUNK OF LAYER 7 HTTP DDOS ATTACKS.**

Keep in mind that the most frequent and common layer 7 DDoS attacks are simply mass amounts of empty and blank GET requests from a variety of IP's, those IP's usually being http/https/socks4/socks5 proxies; therefore setting a ratelimit on as many parts of the website helps stop DDoS attacks much more than you think.

This simple rule above is the first line of defense within the rate limiting system. The next biggest configuration of importance when it comes to DDoS attacks is filtering and only fullfilling requests that do not exceed a certain size threshold. This will be the 2nd line of defense of the rate limiting and it will be discussed more below.


**MANGING EXCESSIVE REQUESTS**

If there are more requests that a client sends than the number of requests allowed defined, or if the shared memory zone becomes full, there needs to be a solution that can make it more efficient so that the least most miniscule amount of resources and data is used to negate/fullfill/block those excessive requests.

    http {
    
    limit_req_zone $binary_remote_addr zone=afzone:10m rate=3r/s;

    server {

        location /index* {
            limit_req zone=afzone burst=5;
            }
        }
    }

The "burst=X;" comment is used to manage how many excessive requests will be stored into the zone. Once the zone is full, excessive requests (in this case the 4th request within the second) will be put into a queue. This queue makes it so that after the 3rd request within that one second is made, the rest of the requests will be non prioritized, with the webserver serving the request when it is comfortable to do so.

Requests above the "burst" or "queue" limit will be given the 503 error. In short, this sets a little wiggle room for visitors to be able to send more requests and not be completely denied. 

    http {
    
    limit_req_zone $binary_remote_addr zone=afzone:10m rate=3r/s;

    server {

        location /index* {
            limit_req zone=afzone burst=5 nodelay;
            }
        }
    }

The "nodelay;" comment in this makes it so that burst requests have no delay or wait time. Any requests that exceed the burst amount will directly be given a 503.


