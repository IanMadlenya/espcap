<h1>espcap</h1>

espcap is a program that uses pyshark to capture packets from a pcap file or live
from a network interface and index them with Elasticsearch.  Since espcap uses
pyshark - which provides a wrapper API to tshark - it can use wireshark dissectors
to parse any protocol.

<h2>Requirements</h2>
<ol>
<li>tshark (included in Wireshark)</li>
<li>pyshark</li>
<li>Elasticsearch client for Python</li>
</ol> 
<h2>Recommendations</h2>

It is highly recommended, although not required, that you use the Anaconda Python 
distribution by Continuum Analytics for espcap. This distribution contains Python
2.7.10 and bundles a rich set of programming packages for analytics and machine 
learning.  You can download Anaconda Python here: http://continuum.io/downloads.

<h2>Installation</h2>
<ol>
<li>Install Wireshark for your OS.</li>
<li>Install pyshark and the Elasticsearch client for Python with pip:
<pre>pip install pyshark
pip install elasticsearch</pre></li>
<li>Create the packet index template by running scripts/templates.sh as follows 
specifying the node IP address and TCP port (usually 9200) of your Elasticsearch 
cluster. If your node IP address is 10.0.0.1 the commands would look like this:
<pre>scripts/templates.sh 10.0.0.1:9200</pre></li>
<li>Set the tshark_path variable in the pyshark/config.ini file.</li>
<li>Run espcap.py as follows to index some packet data in Elasticsearch
<pre>espcap.py --dir=test_pcaps --node=10.0.0.1:9200</pre></li>
<li>Run the packet_query.sh as follows to check that the packet data resides in your
Elasticsearch cluster:
<pre>scripts/packet_query.sh 10.0.0.1:9200</pre></li>
</ol>
<h2>Getting Started</h2>

You run espcap.py as root. If you supply the <tt>--help</tt> flags on the command 
line you'll get the information on the most useful ways to run espcap.py:
<pre>
espcap.py [--dir=pcap_directory] [--node=elasticsearch_host] [--chunk=chunk_size] [--trace]
          [--file=pcap_file] [--node=elasticsearch_host] [--chunk=chunk_size] [--trace]
          [--nic=interface] [--node=elasticsearch_host] [--bpf=packet_filter_string] [--chunk=chunk_size] [--count=max_packets] [--trace]
          [--help]
          [--list-interfaces]

Example command line option combinations:
espcap.py --d=/home/pcap_direcory --node=localhost:9200
espcap.py --file=./pcap_file --node=localhost:9200 --chunk=1000
espcap.py --nic=eth0 --node=localhost:9200 --bpf="tcp port 80" --chunk=2000
espcap.py --nic=en0 --node=localhost:9200 --bpf="udp port 53" --count=500
</pre>
Note that each of these modes is mutually exclusive. If you try to run espcap.py in more 
than one mode you'll get an error message.

You can try espcap.py in file mode using the pcap files contained in test_pcaps. To do 
that run espcap.py as follows (assuming you want to just dump the packets to stdout):
<pre>
espcap.py --dir=./test_pcaps
</pre>
When running in live capture mode you can set a maximum packet count after which the 
capture will stop or you can just hit ctrl-c to stop a continuous capture session. 

espcap uses Elasticsearch bulk insertion of packets. The <tt>--chunk</tt> enables you to set 
how many packets are sent Elasticsearch for each insertion. The default is chunk size is 100,
but higher values (1000 - 2000) are usually better. If you get transport I/O exceptions due
to network latency or an Elasticsearch backend that is not optimally configured, stick with
the default chunk size.

If you want to get more information when exceptions are raised you can supply the <tt>--trace</tt>
flag for either file or live capture modes.

<h2>Packet Indexing</h2>

When indexing packet captures into Elasticsearch, an new index is created for each 
day. The index naming format is <i>packets-yyyy-mm-dd</i>. The date is UTC derived from 
the packet sniff timestamp obtained from pyshark either for live captures or the
sniff timestamp read from pcap files. Each index has two types, one for live capture 
<tt>pcap_live</tt> and file capture <tt>pcap_file</tt>. Both types are dynamically mapped by
Elasticsearch with exception of the date fields for either <tt>pcap_file</tt> or <tt>pcap_live</tt>
types which are mapped as Elasticsearch date fields if you run the templates.sh script
before indexing an packet data.

Index IDs are automatically assigned by Elasticsearch

<h3>pcap_file type fields</h3>

<pre>
file_name          Name of the pcap file from whence the packets were read
file_date_utc      Creation date UTC when the pcap file was created
sniff_date_utc     Date UTC when the packet was read off the wire
sniff_timestamp    Time in milliseconds after the Epoch whne the packet was read
protocol           The highest level protocol
layers             Dictionary containing the packet contents
</pre>

<h3>pcap_live type fields</h3>

The <tt>pcap_live</tt> type is comprised of the same fields except the <i>file_name</i> and
<i>file_date_utc</i> fields.

<h2>Packet Layer Structure</h2>

Packet layers are mapped in four basic sections based in protocol type within each index:
<ol>
<li>Link - link to the physical network media, usually Ethernet (eth).</li>
<li>Network - network routing layer which is always IP (ip).</li>
<li>Transport - transport layer which is either TCP (tcp) or UDP (udp).</li>
<li>Application - high level Internet protocol such as HTTP (http), DNS (dns), etc.</li>
</ol>
Packet layers reside in a JSON section called <tt>layers</tt>. Each of the layers reside in a 
JSON section of the same name. The protocol for each layer is indicated in the <tt>protocol</tt>
field. The highest protocol for the whole packet, which is the application protocol if the 
packet has such a layer, is in the <tt>protocol</tt> field that is at the sam level as the
<tt>layers</tt> section.

Below is an example of an HTTP packet that has been truncated in the places denoted by
<tt><-- SNIP --></tt>. 

<pre>
{
    "_index": "packets-2015-07-30",
    "_type": "pcap_file",
    "_id": "AVAaWs6WtaVU9i_NA6BD",
    "_score": null,
    "_source": {
        "layers": {
            "application": {
                "content_length": "292",
                "_ws_expert": "Expert Info (Chat/Sequence): HTTP/1.1 200 OK\\r\\n",
                "_ws_expert_severity": "2097152",
                "protocol": "http",
                "chat": "HTTP/1.1 200 OK\\r\\n",
                "response_code": "200",
                "content_length_header": "292",
                "server": "Trend Micro 2.5",
                "response_phrase": "OK",
                "response_line": "Server: Trend Micro 2.5\\xd\\xa",
                "connection": "close",
                "last_modified": "Wed, 29 Jul 2015 20:23:38 GMT",
                "request_version": "HTTP/1.1",
                "content_type": "text/html",
                "time": "0.013877000",
                "date": "Thu, 30 Jul 2015 05:22:09 GMT",
                "_ws_expert_group": "33554432",
                "response": "1",
                "cache_control": "max-age=54089",
                "_ws_expert_message": "HTTP/1.1 200 OK\\r\\n",
                "request_in": "4"
            },
            "link": {
                "dst_resolved": "60:f8:1d:cb:43:84",
                "lg": "0",
                "protocol": "eth",
                "addr": "60:f8:1d:cb:43:84",
                "src": "58:23:8c:b4:42:56",
                "addr_resolved": "60:f8:1d:cb:43:84",
                "dst": "60:f8:1d:cb:43:84",
                "type": "2048",
                "src_resolved": "58:23:8c:b4:42:56",
                "ig": "0"
            },
            "network": {
                "checksum_bad": "0",
                "protocol": "ip",
                "checksum_good": "0",
                "ttl": "57",
                "id": "4324",
                "dsfield": "32",
                "addr": "184.51.102.81",
                "proto": "6",
                "flags_rb": "0",
                "dst": "10.0.0.4",
                "version": "4",
                "flags_mf": "0",
                "dsfield_dscp": "8",
                "hdr_len": "20",
                "len": "566",
                "dsfield_ecn": "0",
                "host": "184.51.102.81",
                "frag_offset": "0",
                "src": "184.51.102.81",
                "checksum": "1590",
                "flags_df": "1",
                "dst_host": "10.0.0.4",
                "flags": "2",
                "src_host": "184.51.102.81"
            },
            "transport": {
                "flags_cwr": "0",
                "checksum_bad": "0",
                "protocol": "tcp",
                "seq": "1",
                "flags_ecn": "0",
                "options_type_class": "0",
                "flags_syn": "0",
                "checksum_good": "0",
                "flags_reset": "0",
                "window_size_value": "486",
                "options_type_copy": "0",
                "analysis_initial_rtt": "0.014510000",
                "window_size": "15552",
                "stream": "0",
                "option_len": "10",
                "flags_urg": "0",
                "port": "80",
                "options_timestamp_tsecr": "387044398",
                "window_size_scalefactor": "32",
                "dstport": "59803",
                "hdr_len": "32",
                "options_type_number": "1",
                "len": "514",
                "flags_res": "0",
                "urgent_pointer": "0",
                "analysis_bytes_in_flight": "514",
                "flags_push": "1",
                "flags_ns": "0",
                "flags_ack": "1",
                "option_kind": "8",
                "ack": "285",
                "checksum": "15193",
                "flags_fin": "0",
                "srcport": "80",
                "analysis": "SEQ/ACK analysis",
                "flags": "24",
                "options_timestamp_tsval": "3019033657",
                "nxtseq": "515",
                "options": "01:01:08:0a:b3:f2:cc:39:17:11:d4:2e",
                "options_type": "1"
            },
            "data-text-lines": {
                "envelope": "http",
                "protocol": "data-text-lines"
            }
        },
        "protocol": "http",
        "sniff_timestamp": 1438233730.497605,
        "file_name": "../test_pcaps/test_http.pcap",
        "sniff_date_utc": "2015-07-30 05:22:10",
        "file_date_utc": "2015-09-23 01:41:26"
    }
}
</pre>

The convention for accessing protocol fields in the JSON layers structure is:

<pre>
layers.layer-name.field-name
</pre>

Here are some examples of how to reference specific layer fields taken from the packet JSON shown above:

<pre>
layers.network.protocol          Network layer protocol (always "ip")
layers.network.src               Sender IP address
layers.network.dst               Receiver IP address
layers.transport.protocol        Transport layer protocol (usually either "tcp" or "udp")
layers.transport.srcport         Sender transport port
layers.transport.dstport         Receiver transport port
layers.application.chat          HTTP response
</pre>

Note that some layer protocols span two sections. In the above example, the TCP segment has a <tt>data</tt> 
section associated and the HTTP response has a <tt>media</tt> section. Extra sections like these can be 
associated with their protocol sections by checking the <tt>envelope</tt> field contents.

<h2>Protocol Support</h3>

Technically epscap recognizes all the protocols supported by wireshark/tshark. However, the wireshark
dissector set includes some strange protocols that are not really Internet protocols in the strictest
sense, but are rather parts of other protocols. One example is <tt>media</tt> which is actually used to
label an additional layer for the <tt>http</tt> protocol among other things. espcap uses the protocols.list
to help determine the application level protocol in any given packet. This file is derived from tshark
by running the protocols.sh script in the conf directory. To ensure that espcap has only true Internet
protocols to choose from, the entries in protocols.list that are not truly Internet protocols have
been commented out. Currently the commented out protocols include the following:
<pre>
_ws.expert
_ws.lua
_ws.malformed
_ws.number_string.decoding_error
_ws.short
_we.type_length
_ws.unreassembled
data
data-l1-events
data-text-lines
image-gif
image-jfif
media
null
png
xml
zip
</pre>
If there are any other protocols you believe should not be considered, then you can comment them out in 
this fashion. 

On the other hand If you get a little too frisky and comment out too many protocols or you just want to 
generate a fresh list, you can run the protocols.sh script in the following manner:
<ol>
<li>cd to the conf/ directory</li>
<li>Run the protocols.sh script which produces a clean protocol list in protocols.txt.</li>
<li>Comment out the protocols in the list above and others you don't want to consider.</li>
<li>Replace the contents of protocols.list with the contents of protocols.txt.</li>
</ol>
<h3>Known Issues</h3>
<ol>
<li>File capture mode sometime gets this error when dumping packets to stdout:
<pre>'NoneType' object has no attribute 'add_reader'.</pre></li>
<li>When uploading packet data through the Nginx proxy you may get a <tt>413 Request Entity Too Large</tt> error. This is caused by sending too many packets at each Elasticsearch bulk load call. You can either set the chunk size with the <tt>--chunk</tt> or increase the request entity size that Nginx will accept or both. To set a larger Nginx request entity limit add this line to the http or server or location sections of your Nginx configuration file: 
<pre>client_max_body_size     2M;</pre>
Set the value to your desired maximum entity (body) size then restart Nginx with this command:
<pre>/usr/local/nginx/sbin/nginx -s reload</pre></li>
</ol>
