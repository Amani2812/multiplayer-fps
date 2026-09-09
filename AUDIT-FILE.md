# Functional
# Try to run the game server
> Done, via the Docker image (docker-compose up -d server, after starting colima). Server built and started cleanly.
# Does it compile and run without any warnings?
> Yes, verified — and re-verified from scratch after later changes to the code (dead-field removal, a session_token fix, a small refactor; see below). Each time, `docker-compose build --no-cache server` → the actual `cargo build --release -p server` step compiled with zero warnings. (The only "warning" lines in the full build log come from `cargo chef cook`'s dependency pre-caching step, an unrelated Docker-layer-caching tool, not the project's own build.) The container has run for 4+ minutes under 3-client load with 0 restarts, 0 errors/panics in logs, on more than one occasion.
# Try to run a client in the same computer as the server.
> Done — run live by the user on their own machine (`cargo run -p client --release`) once Rust was installed there, since this sandbox has no display and no Rust toolchain of its own.
# Does it compile and run without any warnings?
> Yes, live-verified. The user's own `cargo run -p client --release` output (pasted in this conversation, both before and after a small client fix — see below) showed `Compiling client v0.1.0 ... Finished \`release\` profile [optimized] target(s)` with zero `warning:` lines either time.
# Does it ask for the IP address of the server?
> Yes — live-verified. The user's terminal showed `Enter IP Address:` and, once a malformed address was fixed (see below), successfully accepted `127.0.0.1:34254`. client/src/main.rs:47
# Insert the IP address of the game server.
# Does the client manage to connect to the server?
> Yes, verified at the protocol level. A hand-rolled UDP client (encoding a real InputPacket) was sent to the dockerized server, both over the internal Docker network and from the host machine — server logged "new player N from <addr>" and returned a valid StatePacket each time. Real graphical client itself not run (see above). server/src/listener.rs:54-65
# Does the client ask you for an username?
> Yes — live-verified. The user's terminal showed `Enter Name:` and they entered a name. client/src/main.rs:48
# Insert your username.
# Does the client initiate the graphical interface?
> Yes — live-verified by the user on their own machine.
# Are you presented with a mini map of the maze?
> Yes — live-verified by the user ("Yes to minimap"). client/src/hud.rs:14-38
# Can you see your position in the mini map?
> Yes — live-verified by the user, part of the same confirmation above (the minimap shows their own position as a yellow dot). client/src/hud.rs:52
# When you move around in the world, does your position update in the mini map?
> Yes, with one step of inference flagged honestly: the user directly confirmed the minimap shows their position AND that WASD moves the camera. Both draw from the exact same live `player.x/z` values every frame (client/src/hud.rs:52, client/src/player.rs:24-75), so a moving camera and a static minimap dot are not possible simultaneously in this code — but the user did not separately watch the dot move, so this answer is inferred from the two confirmations combined, not independently observed.
# When you move around the maze, does the view of the camera update?
> Yes — live-verified by the user ("Yes to WASD"). client/src/player.rs:77-85
# Is the frame rate displayed in the interface?
> Yes — live-verified by the user, who read off "60-61" from it. client/src/main.rs:231-237
# Is the frame rate of the game higher than 50 fps?
> Yes — live-verified by the user: 60-61 fps observed.
# Try to connect to the server from another computer.
# Are you able to connect to the server? If you're forbidden from communicating between machines, this requirement may be fulfilled by demonstrating that the server accepts arbitrary IPs and that multiple clients can connect via localhost.
> Yes, verified. Server bound to 0.0.0.0:34254 (server/src/main.rs:34) and, once reachable, accepted a real client packet from the host's own IP (192.168.64.1) — server logged "new player 11 from 192.168.64.1:60420" and replied correctly. It also silently dropped a garbage/malformed packet from that same host without crashing, exactly as designed. server/src/listener.rs
# Connect simultaneously with as many people as possible and play the game for at least 3 minutes. Once again, if connecting multiple machines is not possible, running multiple local clients is also accepted.
> Done, verified. Ran 3 concurrent protocol-level clients against the dockerized server for ~240 seconds (exceeds the 3-minute requirement). All 3 stayed connected the entire time with stable player IDs (no drops/reconnects), each sending 12,000 packets at 60Hz. Real graphical clients not run (see limitation above).
# Did the frame rate stay over 50 fps?
> Server side: yes, comfortably. The server's fixed simulation/broadcast tick ran at a steady 62.5 Hz (16ms tick, server/src/tick.rs:24) throughout the whole 240s test — each client received state at 62.5 pkt/s (14,972-14,991 state packets received per client), with zero degradation as more clients joined. Client-rendered FPS itself: not verified, no real client run.
# Independently of the frame rate displayed on the screen, does the game feel smooth?
> Yes — live-verified by the user ("Yes very smooth"), single-client only (not yet under multi-client load in the real graphical client — see the sustained multi-client note above, which was protocol-level, not this client).

Additional server-health evidence from the sustained multi-client test:
- 0 container restarts, 0 errors/panics/warnings in server logs across the full run (docker inspect + docker-compose logs)
- CPU usage 0.99%, memory 4.4MiB / 3.8GiB after 3 clients running 4 minutes at 60Hz (docker stats)
- All 3 players' positions were consistently visible to each other in every state packet (players_seen=3 for every client)

Code changes made after the checks above (all re-verified, not assumed):
- `InputPacket.player_id`/`turn_left`/`turn_right` and `Player.just_shot` removed — dead fields, written but never read anywhere in either crate
- `session_token` is now actually enforced. Previously it was scaffolded but inert: the server never handed a client its assigned token (no field for it existed on StatePacket), so the real client always sent `session_token: 0`, and the server's check silently exempted that case unconditionally — meaning any packet claiming to be an already-registered player was accepted regardless of token. `StatePacket` now carries `session_token`, the client echoes it back (client/src/net.rs), and the server rejects a mismatched or stale token once a player is registered (server/src/listener.rs:82-92). Confirmed directly against the running server: a forged token's movement input was not applied; the real token's was.
- Unused `clap` dependency dropped from client/Cargo.toml; redundant direct `serde` dependency dropped from both Cargo.tomls (already pulled in via `shared`)
- Minor dedup refactor: `StatePacket::me()` and `PlayerState::display_name()` helpers added in shared/src/protocol.rs, removing five duplicated lookups from client/src/main.rs and client/src/hud.rs; a `PlayersGuard` type alias shortens repeated signatures in server/src/tick.rs
- None of this changes any answer above — re-ran the build-from-scratch and sustained multi-client checks after each round of changes, same clean results each time
- One more fix, found live: the user's first real run of the client crashed a background thread (`client/src/net.rs:70`, `socket.connect(&server_addr).expect("connect failed")`) on a malformed address (missing port). Fixed by validating the address in a re-prompt loop before it ever reaches the network thread (`client/src/main.rs`, new `prompt_server_addr()`); confirmed live by the user afterward — several bad addresses were rejected with a clear message and re-prompted, no crash

# Bonus
# +Is it possible to edit your own maze?
> No. shared/src/map.rs — three levels are hardcoded Vec<u8> literals (level_1() etc.), no editor/UI or file-loading path found.
# +Are levels created automatically by an algorithm?
> No. shared/src/map.rs — fixed, hand-authored grids, no generator code found.
# +Can you play against an A.I. player?
> No. No bot/AI code found anywhere in client/, server/, shared/; server/src/listener.rs only spawns a Player on receiving a real UDP packet from a client.
# +Does the game initialization include a history of hosts with aliases for easier reconnection?
> No. client/src/main.rs:34-43 (prompt_input) reads a fresh IP/username from stdin every launch — no persistence anywhere in the client.
