FROM kibana:latest
RUN cd /opt/kibana
RUN gosu kibana /opt/kibana/bin/kibana plugin --install elastic/sense
ADD ./conf/kibana.yml /opt/kibana/config/kibana.yml
