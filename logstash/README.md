# Logstash

We've provided a default logstash.conf that handles accepting and parsing out ClearBlade Platform log statements to useful fields within elasticsearch. This includes timestamp, log level, and message. This config also handles incoming metricbeat data, and sends it to a seperate index within elasticsearch

