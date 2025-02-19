FROM alpine:3.21

EXPOSE 514
VOLUME /etc/rsyslog.conf.d
RUN apk --no-cache update && apk add rsyslog --no-cache
CMD ["/usr/sbin/rsyslogd", "-n", "-f", "/etc/rsyslog.conf.d"]