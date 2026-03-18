# See https://aka.ms/customizecontainer to learn how to customize your debug container and how Visual Studio uses this Dockerfile to build your images for faster debugging.

# This stage is used when running from VS in fast mode (Default for Debug configuration)
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
USER $APP_UID
WORKDIR /app
EXPOSE 8080
EXPOSE 8081


# This stage is used to build the service project
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
ARG BUILD_CONFIGURATION=Release
ARG VERSION=1.0.0
ARG SKIP_EMBEDDINGS=true
WORKDIR /src
COPY ["csla-mcp-server/csla-mcp-server.csproj", "csla-mcp-server/"]
RUN dotnet restore "./csla-mcp-server/csla-mcp-server.csproj"
COPY . .
WORKDIR "/src/csla-mcp-server"
RUN dotnet build "./csla-mcp-server.csproj" -c $BUILD_CONFIGURATION -o /app/build /p:InformationalVersion=$VERSION

# Generate the embeddings file if SKIP_EMBEDDINGS is false, otherwise create an empty file to avoid build errors in the final stage
RUN echo "[]" > /embeddings.json
# if SKIP_EMBEDDINGS is false, run the script to generate the embeddings file
RUN if [ "$SKIP_EMBEDDINGS" = "false" ]; then \
      echo "Building embeddings generator..." \
      dotnet build csla-embeddings-generator/csla-embeddings-generator.csproj -c Release \
      echo "Generating embeddings..." \
      dotnet run --project csla-embeddings-generator/csla-embeddings-generator.csproj --configuration Release -- --examples-path ./csla-examples --output /embeddings.json \
    else \
      echo "Skipping embeddings generation..."; \
    fi

# This stage is used to publish the service project to be copied to the final stage
FROM build AS publish
ARG BUILD_CONFIGURATION=Release
ARG VERSION=1.0.0
RUN dotnet publish "./csla-mcp-server.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false /p:InformationalVersion=$VERSION

# This stage is used in production or when running from VS in regular mode (Default when not using the Debug configuration)
FROM base AS final
WORKDIR /
COPY --from=build /src/csla-examples /csla-examples
WORKDIR /app
COPY --from=publish /app/publish .
# Copy pre-generated embeddings (created by build.sh script)
COPY --from=build /embeddings.json ./embeddings.json
ENTRYPOINT ["dotnet", "csla-mcp-server.dll"]