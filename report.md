# Meassuring Churn in P2P Botnets

## Problem: Churn

## Existing System: BMS

The Botnet Monitoring System (BMS) is a platform to crawl and monitor P2P botnets and building statistics over time [^bms].
Currently the following botnets are being observed:

* DDG: a cryptocurrency mining botnet (Monero), mostly aimed at database servers [^ddg] [^ddg_netlab].
* Hajime: IoT botnet, targeting mostly the same unsecured/badly secured IoT devices as Mirai [^hajime]
* Hide 'n' Seek: IoT devices and misconfigured database servers (e.g. MongoDB, CouchDB), cryptocurrency mining [^hns]
* Mozi: IoT, DDoS, data exfiltration [^mozi]
* Sality: Windows, file infector, spam, proxy, data exfiltration, DDoS [^sality]

A requirement of the task was to integrate the implemented solution into BMS, allowing to build a network graph on which analysis can be performed.

## Implementation

The solution was implemented as a scheduler in Python, running in a Docker container, that would periodically query replies we received from bots and would calculate the online sessions for each bot.

A session is defined as a consecutive series of 5 minute time buckets in which a peer is confirmed to be online. If a peer hasn't responded for more than 15 minutes, the session is considered closed and a new session starts, if the peer is seen again.

We recorded sessions both based on IP address + port and the unique bot ID, the peer uses inside the network.
Major differences in those sessions would be an indicator for either more than one peer using the same IP (e.g. devices behind a NAT) or really long running nodes using a residential internet connection that rotates IP addresses periodically.

### Problems

Most problems we encountered, were performance problems. The solution was correct on small test datasets but did not perform well or at all on realworld data.

Finding sessions by querying timeframes manually and selecting distinct `IP + Port` combinations in those buckets resulted in slow runtime since a roundtrip to the database was needed for each frame.

```
SELECT bot_id, ip, port, botnet_id
FROM bot_edges
WHERE time_seen BETWEEN %(start)s AND %(start)s + %(frame)s
```

With this query, a loop was needed to query each frame.
The PostgreSQL extension `Timescale` offers a function `time_bucket` [^time_bucket] to perform an equivalent operation on the database, which allows to query many time frames at once and gets rid of the rountdtrips.

```
SELECT time_bucket(%(bucket_size)s, time_seen), bot_id, ip, port, botnet_id
FROM bot_edges
WHERE br.time_seen >= %(start)s
```

## References

[^bms]: [Poster: Challenges of Accurately Measuring Churn in P2P Botnets](https://dl.acm.org/doi/10.1145/3319535.3363281) (Böck, Leon and Karuppayah, Shankar and Fong, Kory and Mühlhäuser, Max and Vasilomanolakis, Emmanouil)
[^ddg]: [DDG (Malware Family)](https://malpedia.caad.fkie.fraunhofer.de/details/elf.ddg) (Fraunhofer-Institut für Kommunikation, Informationsverarbeitung und Ergonomie FKIE)
[^ddg_netlab]: [DDG: A Mining Botnet Aiming at Database Servers](https://blog.netlab.360.com/ddg-a-mining-botnet-aiming-at-database-servers/) ([https://netlab.360.com/](Network Security Research Lab at 360))
[^hajime]: [Measurement and Analysis of Hajime, a Peer-to-peer IoT Botnet](https://par.nsf.gov/servlets/purl/10096257) (Herwig, Stephen and Harvey, Katura and Hughey, George and Roberts, Richard and Levin, Dave and University of Maryland and Max Planck Institute for Software Systems (MPI-SWS))
[^hns]: [Hide ‘N Seek botnet continues infecting devices with default credentials, building a P2P network and more](https://blog.avast.com/hide-n-seek-botnet-continues) (Avast)
[^mozi]: [Mozi, Another Botnet Using DHT](https://blog.netlab.360.com/mozi-another-botnet-using-dht/) (Turing, Alex and Wang, Hui)
[^sality]: [Sality: Story of a Peer-to-Peer Viral Network](https://web.archive.org/web/20120403180815/http://www.symantec.com/content/en/us/enterprise/media/security_response/whitepapers/sality_peer_to_peer_viral_network.pdf) (Falliere, Nicolas)
[^time_bucket]: [Timescale `time_bucket()`](https://docs.timescale.com/api/latest/hyperfunctions/time_bucket/)
