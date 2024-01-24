FROM mysql:8.0.33
LABEL "Project"="Vprofile"
LABEL "Author" = "me"

ENV MYSQL_ROOT_PASSWORD="12345"
ENV MYSQL_DATABASE="accounts"

ADD db_backup.sql docker-entrypoint-initdb.d/db_backup.sql
EXPOSE 3306