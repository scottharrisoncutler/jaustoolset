# Dockerfile for safely building and running the Conway Automaton
# in an isolated container environment.
#
# Build:  docker build -t automaton .
# Run:    docker run -it --name my-automaton automaton
#
# See AUTOMATON_DOCKER_GUIDE.md for full instructions and security notes.

# ---------------------------------------------------------------------------
# Build stage — compile TypeScript and install dependencies
# ---------------------------------------------------------------------------
FROM node:20-slim AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    python3 \
    make \
    g++ \
    ca-certificates \
  && rm -rf /var/lib/apt/lists/*

RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /build

RUN git clone --depth 1 https://github.com/Conway-Research/automaton.git .

RUN pnpm install --frozen-lockfile || pnpm install

RUN pnpm build

# ---------------------------------------------------------------------------
# Runtime stage — smaller image, fewer attack-surface packages
# ---------------------------------------------------------------------------
FROM node:20-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    ca-certificates \
  && rm -rf /var/lib/apt/lists/*

RUN corepack enable && corepack prepare pnpm@latest --activate

# Run as a non-root user for additional isolation
RUN groupadd -r automaton && useradd -r -g automaton -m -d /home/automaton automaton

WORKDIR /app

COPY --from=builder /build /app

RUN chown -R automaton:automaton /app

USER automaton

# The automaton persists state in ~/.automaton/
VOLUME ["/home/automaton/.automaton"]

EXPOSE 3000

ENTRYPOINT ["node", "dist/index.js"]
CMD ["--run"]
