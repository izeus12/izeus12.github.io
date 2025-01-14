# 9447's CTF 2014

## On Redis & AES Encryption 

### The Client File

The first file was a script **client.py**, where, by using Python's [socket](https://docs.python.org/2/library/socket.html) library, showed how a connection to the server could be made:

```py
import os, socket, struct, sys
from Crypto.Cipher import AES

class EncryptedStream(object):
 key = 'this is not the flag nor the key'[:16]
 def __init__(self, host, port):
 self.sock = socket.socket()
 self.sock.connect((host, port))
 def send(self, msg):
 while len(msg) % 16:
 msg += '\0'
 iv = os.urandom(16)
 aes = AES.new(self.key, AES.MODE_ECB, iv)
 enc = aes.encrypt(msg)
 self.sock.send(struct.pack('<I', len(enc)))
 self.sock.send(enc)
 def recv(self, nbytes):
 return self.sock.recv(nbytes)

client = '''\
HELLO
SHOW VERSION
SET example This tiny script is basically a RedisStore...
GET example
SHOW KEYS
SET brucefact#1 Bruce Schneier can break elliptic curve cryptography by bending it into a circle
SET brucefact#2 Bruce Schneier always cooks his eggs scrambled. When he wants hardboiled eggs, he unscrambles them
SET brucefact#3 Bruce Schneier could solve this by inverting md5 hash of the flag
ENCRYPTION HEX
MD5 flag
'''

stream = EncryptedStream(sys.argv[1], int(sys.argv[2]))
stream.send(client)
while 1:
 data = stream.recv(1000)
 if not data: break
 sys.stdout.write(data)
```

This client script makes [AES](http://en.wikipedia.org/wiki/Advanced_Encryption_Standard) encrypted packets for a given host and port (the arguments), with the class **EncryptedStream**. It then sends the packets and prints out any received stream.

The snippet also shows an example of a client packet, with some request options (which we will see the response later in the network dump).


### The Server File

The second file was the **server.py** script, which is a [Redis](http://redis.io/) like a database (hence, the *nosql* title). Unlike SQL databases, Redis *maps keys to types of values*. In this challenge, the idea was to recover an entry that had the key **flag** returning the value of the flag.

In the script below, besides creating this database, functions such as: **AES decrypting** (encryption), **MD5** (hashing), and **hex** (encoding) are implemented using Python's library:

```py
import hashlib, os, signal, struct, sys
from Crypto.Cipher import AES

key = 'this is not the flag nor the key'[:16]
db = { }

def md5(data):
 return hashlib.md5(data).digest()

def decrypt(data):
 iv = os.urandom(16)
 aes = AES.new(key, AES.MODE_ECB, iv)
 data = aes.decrypt(data)
 return data.rstrip('\0')

def reply_plain(message):
 sys.stdout.write(message + '\n')

def reply_hex(message):
 # This is totally encrypted, right?
 sys.stdout.write(message.encode('hex') + '\n')

def main():
 global db
 reply = reply_plain

 datalen = struct.unpack('<I', sys.stdin.read(4))[0]
 data = ''
 while len(data) != datalen:
 s = sys.stdin.read(1)
 if not s:
 sys.exit(1)
 data += s
 data = decrypt(data)

 commands = data.split('\n')

 for cmd in commands:
 if not cmd:
 continue
 if ' ' in cmd:
 cmd, args = cmd.split(' ', 1)

 if cmd == 'HELLO':
 reply('WELCOME')
 elif cmd == 'SHOW':
 if args == 'VERSION':
 reply('NoRedisSQL v1.0')
 elif args == 'KEYS':
 reply(repr(db.keys()))
 elif args == 'ME THE MONEY':
 reply("Jerry, doesn't it make you feel good just to say that!")
 else:
 reply('u w0t m8')
 elif cmd == 'SET':
 key, value = args.split(' ', 1)
 db[key] = value
 reply('OK')
 elif cmd == 'GET':
 reply(args + ': ' + db.get(args, ''))
 elif cmd == 'SNIPPET':
 reply(db[args][:10] + '...')
 elif cmd == 'MD5':
 reply(md5(db.get(args, '')))
 elif cmd == 'ENCRYPTION':
 if args == 'HEX':
 reply = reply_hex
 reply('OK')
 elif args == 'OFF':
 reply = reply_plain
 reply('OK')
 else:
 reply('u w0t m8')
 else:
 reply('Unknown command %r' % (cmd))


if __name__ == '__main__':
 signal.alarm(10)
 signal.signal(signal.SIGALRM, lambda a,b: sys.exit(0))
 main()
```

This script pretty much gives away all the requests that you can issue to inspect the database.

In addition, a crucial detail is to understand how the client encrypts the commands using the [electronic codebook (ECB)](http://en.wikipedia.org/wiki/Block_cipher_mode_of_operation#Electronic_codebook_.28ECB.29) block cipher type. In this type of operation the message is divided into blocks that are encrypted separately ([PyCryptos's AES.MODE_ECB](https://www.dlitz.net/software/pycrypto/api/2.6/)).



### The PCAP File

The last file was a **pcap** dump. When opening it with [Wireshark](http://bt3gl.github.io/wiresharking-for-fun-or-profit.html), I verified it was really short, and the content was simply a [TCP handshake](http://www.inetdaemon.com/tutorials/internet/tcp/3-way_handshake.shtml). Right-clicking some packet and selecting *Follow TCP Stream* returned the dump of the connection suggested by the **client.py** script:

![cyber](http://i.imgur.com/2Y6aaW1.png)

However, we see that the database has already an entry for flag:

```
['flag', 'example']
```

The response **4f4b** is **OK** in ASCII, meaning that the switch **ENCRYPTION HEX** was on (it's good to keep in mind that the "encryption" is actually just an encoding in hex, *i.e*, completely reversible).

Finally, our MD5 for the flag was printed as **b7133e9fe8b1abb64b72805d2d97495f**.

As it was expected, searching for this hash in the usual channels (for example [here](http://hash-killer.com/), [here](http://www.md5this.com/), or [here](http://www.hashkiller.co.uk/)) was not successful: *brute force it is not the way to go*.


### Solving the Challenge

It's pretty clear from our **server.py** script that we could craft a direct request to the server to get our flag before it is hashed to MD5. For example, if the request *GET flag*,

```py
 elif cmd == 'GET':
 reply(args + ': ' + db.get(args, ''))
```

is exactly like *MD5 flag*, without the hashing:

```py
 elif cmd == 'MD5':
 reply(md5(db.get(args, '')))
```

However, we do not have the AES key used by the server, only an example of communication given by the PCAP file. How do we get to send a **GET flag** message?

The first thing that comes to our minds is to use the network dump to replay the message, re-shaping it somehow to have a *GET flag*. Remember that the blocks have a size of 16, and we see two blocks that are particularly interesting:

```
ION HEX
MD5 flag
```

and

```
edisStore...
GET
```

Now we check how the oracle responds to several types of responses:

```
$ python client.py 54.148.249.150 4479
```
We are able to learn that if we send a **command without arguments** or an **invalid command**, the argument variables (*args*) is not overwritten: it gets the **same args value from the previous valid request**! That's wonderful!

Now the solution is clear:

1. We send the invalid command and a valid command with the argument that we will keep: ```ION HEX\nMD5 flag```.
2. We send the invalid command and command without an argument: ```edisStore...\nGET``` (this will get the last valid argument (*flag*), returning us the flag!).
 
