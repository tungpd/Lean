# Use QuantConnect Research as the base
FROM quantconnect/research:latest

# Install dos2unix utility and Node.js 20.x (required for SonarQube JSON analysis)
RUN apt-get update && apt-get install -y curl dos2unix gnupg && \
    curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \
    apt-get install -y nodejs && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    pip install --no-cache-dir quantconnect-stubs