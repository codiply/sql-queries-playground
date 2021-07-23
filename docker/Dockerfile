FROM postgres:9.6.22-buster

RUN  apt-get update \
  && apt-get install -y \
     wget \
     unzip \
  && rm -rf /var/lib/apt/lists/*

# ARG DATA_SIZE=small
ARG DATA_SIZE=medium
# ARG DATA_SIZE=big

RUN wget https://edu.postgrespro.com/demo-${DATA_SIZE}-en-20170815.zip -O /tmp/demo.zip
RUN unzip /tmp/demo.zip
RUN sed -i 's/DROP DATABASE demo;//g' *.sql
RUN mv *.sql /docker-entrypoint-initdb.d/