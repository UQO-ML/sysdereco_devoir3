FROM stain/jena-fuseki:5.1.0

WORKDIR /fuseki

COPY fuseki_config.ttl /fuseki/config.ttl
COPY ontology /fuseki/data
RUN mkdir -p /fuseki/databases/recommendations

EXPOSE 3030

CMD ["sh", "-c", \
  "/jena-fuseki/fuseki-server --config=/fuseki/config.ttl"]
