# 380-chat

### Shein Htike
In this code, the chat app first uses triple Diffie-Hellman to initally exchange keys over the websocket.
Both parties require only the other party's public key.

After a key exchange, both parties should now have a symmetric key.
After this, all messages are encrypted and decrypted using EVP_aes_256_gcm().

For EVP_aes_256_gcm, a 64 bit counter is used as the IV or nonce in order to prevent replay attacks and
ensure that the same message looks different when sent multiple times.

Additionally, the GCM mode provides it's own GMAC or **Galois Message Authentication Code**. This means 
that no HMAC is required for message authentication purposes.

In summary, this protocol:

* **Has forward secrecy** - A unique derived key is used for every connection.
* **Has mutual authentication** - Triple diffie-hellman key exchange ensures that only the person with the correct secret
key can get the same KDF output.
* **Is resilient against replay attacks** - A nonce counter prevents replaying while using minimal memory compared to a random nonce.
* **Has message authentication** - EVP_aes_256_gcm provides its own message authentication in the algorithm

Security flaws:

* I did not implement any handling for messages that are too long.
* It is possible that the program has a memory leak somewhere
* Situations such as invalid session keys simply cause the program to halt without wiping RAM.
* A more secure RNG should be used for both long term and ephemeral key generation.


If I had more time, those flaws could have been addressed.

### MacOS Compatibility

I had to make a number of tweaks to get the code to run on macOS:

1. I ran `brew install gtk+3 openssl gmp` to install all dependencies using homebrew
2. In my makefile, I included openssl and gmp in my `pkg-config` flags
```makefile
LDADD    := -lpthread -lcrypto $(shell pkg-config --libs gtk+-3.0 openssl gmp
INCLUDE  := $(shell pkg-config --cflags gtk+-3.0 openssl gmp)
```
3. I found a file `endian.h` on github which acted as a subsitute for `<endian.h>` and defined some missing symbols.
4. For some reason HOST_LIMIT_MAX is not defined anywhere on macOS so I had to do it manually inside chat.c
```c
#ifndef HOST_NAME_MAX
#define HOST_NAME_MAX 255
#endif
```
5. The following code refused to run on macOS. `%ms` does not seem to be supported. I even tried compiling with `gcc-14` instead of the default clang compiler but I ran into the same issue.
```c
	if (fscanf(f,"name:%ms\n",&name) != 1) {
		rv = -2;
		goto end;
	}
```
To fix this, I re-implemented something equivalent using getline() but it is probably less secure.
```c
    char *line = NULL;
    size_t len = 0;
    ssize_t read = getline(&line, &len, f);
    if (read == -1) {
        free(line);
		rv = -2;
		goto end;
    }
    if (read <= 5 || memcmp(line,"name:",5) != 0) {
        rv = -2;
        free(line);
        goto end;
    }
    int i;
    bool newLineFound = false;
    for(i = 0; i < read; i++){
        if(line[i] == '\n'){
            line[i] = 0;
            newLineFound = true;
            break;
        }
    }
    if(!newLineFound){
        rv = -2;
        free(line);
        goto end;
    }
    size_t namelen = strlen(line+5);
    name = malloc(namelen+1);
    memcpy(name,line+5,namelen+1);
    free(line);

```