FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY Monday.sln Monday.sln
COPY WebApi/Monday.WebApi.csproj WebApi/Monday.WebApi.csproj
RUN dotnet restore

COPY . .

RUN dotnet build WebApi/Monday.WebApi.csproj --configuration Release --output /app/build

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app

RUN apt-get update && apt-get install -y curl=7.88.1-10+deb12u5 --no-install-recommends && rm -rf /var/lib/apt/lists/*

RUN groupadd -g 10000 dotnet && useradd -u 10000 -g dotnet dotnet && chown -R dotnet:dotnet /app
USER dotnet:dotnet

ENV ASPNETCORE_URLS http://*:5080
EXPOSE 5080

COPY --chown=dotnet:dotnet --from=build /app/build .

HEALTHCHECK --interval=5s --timeout=10s --retries=3 CMD curl --fail http://localhost:5080/health || exit 1

ENTRYPOINT ["dotnet", "Monday.WebApi.dll"]