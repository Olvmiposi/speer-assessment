# speer-assessment
Asynchronous TTL Expiration Management
This DNS API service supports optional TTL (Time-To-Live) values for DNS records. TTL is handled asynchronously using a background cleanup job.

🔧 How It Works
Each DNS record (A or CNAME) can optionally be assigned a TTL value (in seconds) upon creation.

Internally, the system tracks TTLs using a Map within each record, keyed as:
TTL["A|192.168.1.1"] = <expiry_timestamp>
TTL["CNAME|target.com"] = <expiry_timestamp>

An asynchronous background worker runs on a fixed schedule (setInterval) every 60 seconds:

It checks all TTL entries.

It removes any expired records.

If a hostname has no remaining records, it deletes the hostname from memory.

🛠 Implementation Details
Worker Type: Native JavaScript setInterval

Frequency: Every 60 seconds (CLEANUP_INTERVAL_MS)

Expiry Logic: TTLs are stored as absolute Unix timestamps (in milliseconds) and compared to Date.now()

Memory Cleanup: Hostnames with zero records are fully purged to avoid memory bloat.

✅ Example Usage
Add an A record with TTL:
 ```
POST /api/dns
{
  "type": "A",
  "hostname": "temp.com",
  "value": "1.2.3.4",
  "ttl": 300
}]
 ```
This record will automatically expire and be removed after 5 minutes (300 seconds).

CNAME Resolution & Loop Detection
This DNS API supports realistic CNAME behavior, including CNAME chaining and loop prevention, adhering to DNS protocol constraints.

🔍 How CNAME Resolution Works
When resolving a hostname:

The resolver checks if there are A records.

If not, it checks for a CNAME.

If a CNAME exists, it follows the target hostname recursively.

This continues until an A record is found or a loop is detected.

🔄 Loop Detection Strategy
A Set of visited hostnames (seen) is maintained during resolution.

If a hostname reappears during the recursive chain, an error is thrown:

 ```
{
  "error": "CNAME loop detected"
}
 ```
This prevents infinite resolution chains and enforces DNS integrity.

✅ Example CNAME Chain
Given the following records:

 ```
POST /api/dns
{
  "type": "CNAME",
  "hostname": "one.com",
  "value": "two.com"
}
 ```
 ```
POST /api/dns
{
  "type": "CNAME",
  "hostname": "two.com",
  "value": "three.com"
}
 ```
 ```
POST /api/dns
{
  "type": "A",
  "hostname": "three.com",
  "value": "192.168.1.1"
}
 ```
A query to /api/dns/one.com will resolve all the way to 192.168.1.1.

🚫 Example Loop Detection
 ```
POST /api/dns
{
  "type": "CNAME",
  "hostname": "a.com",
  "value": "b.com"
}
 ```
 ```
POST /api/dns
{
  "type": "CNAME",
  "hostname": "b.com",
  "value": "a.com"
}
Querying /api/dns/a.com will return:

 ```
 ```
{
  "error": "CNAME loop detected"
}
 ```
