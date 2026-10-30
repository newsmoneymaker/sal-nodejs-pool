# Needs librandomx (https://github.com/tevador/RandomX) and OpenSSL headers. Salvium mines stock, unmodified
# rx/0 (same as Monero/Qwertycoin), so this MUST be built against Salvium's own external/randomx submodule
# (src/build/release/external/randomx/librandomx.a + src/external/randomx/src/randomx.h) -- never reuse another
# coin's prebuilt rxhash/librandomx, since a coin whose algo is "rx/<name>" (epic, arq, c64, keva's RandomKeva...)
# typically has tweaked RandomX constants and will silently compute wrong hashes for a stock rx/0 coin.
# RANDOMX=/path/to/prefix make        (prefix has include/randomx.h and lib/librandomx.a)
RANDOMX ?= /usr/local
CXX ?= g++
rxhash: rxhash.cpp
	$(CXX) -O2 -std=c++11 -pthread -I$(RANDOMX)/include -o $@ $< $(RANDOMX)/lib/librandomx.a -lcrypto
clean:
	rm -f rxhash
