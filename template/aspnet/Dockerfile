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
COPY function/tests/integration/integration.csproj ./tests/integration/
COPY function/tests/unit/unit.csproj ./tests/unit/

# list files in WORKDIR
RUN ls -R /source/faas/function

# restore packages
COPY add-nuget-config.sh /tmp/
RUN chmod u+x /tmp/add-nuget-config.sh
RUN --mount=type=secret,id=nuget.config /tmp/add-nuget-config.sh nuget.config \
    dotnet restore

# Copies the rest of the code
WORKDIR /source/faas
COPY . .

# list files in WORKDIR after copying the rest of the code
RUN ls -R /source/faas

# Run the unit tests
RUN dotnet test function/tests/unit/unit.csproj -c release

# build and publish
RUN dotnet publish \
    -c release \
    -r linux-x64 \
    --no-self-contained \
    -p:PublishReadyToRun=true \
    -o published \
    function/src/function.csproj

# list files in the published directory
RUN ls -R /source/faas/published

# runner
FROM mcr.microsoft.com/dotnet/aspnet:8.0-bookworm-slim

COPY --from=watchdog /fwatchdog /usr/bin/

# Check if the 'app' group exists, create it if it doesn't
RUN if getent group app > /dev/null 2>&1; then \
    echo "Group 'app' already exists"; \
    else \
    addgroup --system app; \
    fi \
    && adduser --system --ingroup app appuser

WORKDIR /home/app/
COPY --from=builder /source/faas/published .
COPY --from=builder /source/faas/function/src/*.json . # Ensure the JSON files are copied
RUN chown app:app -R /home/app

# list files in the app directory after copying
echo "yido testing..."
RUN ls -R /home/app

USER app

ENV fprocess="./function"
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
