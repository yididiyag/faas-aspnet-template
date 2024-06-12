# syntax=docker/dockerfile:1.3
FROM ghcr.io/openfaas/of-watchdog:0.9.6 as watchdog

FROM mcr.microsoft.com/dotnet/sdk:8.0-bookworm-slim as builder
ARG RUN_TESTS="true"
ARG AZURE_DEVOPS_FEED_USERNAME
ARG AZURE_DEVOPS_FEED_ACCESSTOKEN

RUN apt update && apt install libxml2-utils -y

# suppress data collection
ENV DOTNET_CLI_TELEMETRY_OPTOUT 1

# caches restore result by copying csproj files separately
WORKDIR /source/faas/function
COPY function/*.sln .
COPY function/src/function.csproj ./src/

# list files in WORKDIR
RUN ls -a

# restore packages
COPY add-nuget-config.sh /tmp/
RUN chmod u+x /tmp/add-nuget-config.sh
RUN --mount=type=secret,id=nuget.config /tmp/add-nuget-config.sh nuget.config \
    dotnet restore

# Copies the rest of the code
WORKDIR /source/faas
COPY . .

# build and publish
RUN dotnet publish \
    -c Release \
    -o published \
    function/src/function.csproj

# runner
FROM nginx:alpine
COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=builder /source/faas/published/wwwroot /home/www-data/app

WORKDIR /home/app/

RUN ls -a

ENV fprocess="nginx"
ENV cgi_headers="true"
ENV mode="http"
ENV upstream_url="http://localhost:9000"
ENV ASPNETCORE_URLS=http://+:9000

ENV exec_timeout="120s"
ENV write_timeout="120s"
ENV read_timeout="120s"

ENV prefix_logs="false"

EXPOSE 8080

HEALTHCHECK --interval=3s CMD [ -e /tmp/.lock ] || exit 1

CMD ["fwatchdog"]