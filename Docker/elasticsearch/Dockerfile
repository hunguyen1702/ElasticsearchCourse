FROM elasticsearch:latest
RUN cd /usr/share/
RUN bin/plugin install license
RUN bin/plugin install marvel-agent
ADD ./conf/elasticsearch.yml /usr/share/elasticsearch/config/elasticsearch.yml
