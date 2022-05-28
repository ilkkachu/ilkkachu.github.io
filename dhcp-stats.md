Collecting DHCP statistics with 'dhcpd-pools' and 'rrdtool'

[`dhcpd-pools`](http://dhcpd-pools.sourceforge.net/) is a nifty tool for
getting statistics about DHCP pool usage on ISC dhcpd.  Also
[RRDtool](https://oss.oetiker.ch/rrdtool/) is rather nice in collecting data
this kind of numeric data and drawing graphs from it.

The default output shows stats for all pools separately, along with
collected stats for each shared network. I'm only interested in the
shared networks, though, and the `-L22` argument can be used to tell
it to show just that. The output looks like this:

```
# dhcpd-pools -L22          
Shared networks:
name                   max   cur     percent  touch    t+c  t+c perc
net-this              1007   351     34.856     656   1007   100.000
net-that               748   253     33.824     492    745    99.599
[...]
```

(The numbers are rather low since it's summer time and many students
go back home or somewhere else for work.)

This is the point where we probably should have some logic for sending
warnings if the usage goes over some limit. Though that should probably come
with some smarts, since it doesn't make sense to send an email warning about
every time the data is collected (be it every 15 mins or every hour). Maybe
just check the data later and to see if the daily maximum goes over a limit.


But for now, let's settle on just collecting the data.  For that, I created
an RRDtool database for each of the shared networks, with the data sources
`used`, `avail`, and `pct`, for the addresses in use right now, total
addresses available in the network, and the usage percentage (which could be
calculated from the other two, but dhcpd-pools gives it directly, and
storing it here makes it a bit simpler when reading the data).  These
correspond to `cur`, `max` and `percent` in dhcpd-pools output.

Here's a simple script to update the RRD files, with the data acquired over
SSH from the DHCP server:

```
now=$(date +%s)
while read name max cur pct rest; do
    if [[ $name = net* ]]; then          # ignore the header lines
        name=${name#net-}                # remove the net- prefix
        rrdfile=dhcpstats-"$name".rrd
        if [[ -e "$rrdfile" ]]; then     # ignore networks with no rrd file
            rrdtool update "$rrdfile" -t used:avail:pct "$now:$cur:$max:$pct"
        fi             
    fi
done < <(ssh -i "$identity_file" "$username@$dhcpserver" dhcpd-pools -L22)
```

Though this is a case for using SSH keys with a forced command, so the key
for getting the statistics can more safely be left without a passphrase.
In `authorized_keys`:

```
command="dhcpd-pools -L22",restrict,from="monitorhost.example.org" ssh-rsa AAAAB3...F6mJdX dhcp stats key
```

Using `-L02` with `dhcpd-pools` would have it omit the header lines, but I
wanted to keep them there in case someone looks at the data using something
other than this script and wonders what the numbers are.

I built graphs of the usage statistics with a simple script to make it
easier to create multiple graphs with separate timeframes. The script is
[graph.sh].

To check if the usage goes too high, one option would be to look at the
daily maximum after each day. E.g. 


```
rrdtool fetch --align-start --resolution 86400 -e now-1d -s now-1d "$rrdfile" MAX
```

with some awk/shell glue on top to send a warning email.



