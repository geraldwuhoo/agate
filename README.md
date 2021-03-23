# Agate

## Simple Gemini server for static files

Agate is a server for the [Gemini] network protocol, built with the [Rust] programming language. Agate has very few features, and can only serve static files. It uses async I/O, and should be quite efficient even when running on low-end hardware and serving many concurrent requests.

Since Agate by default uses port 1965, you should be able to run other servers (like e.g. Apache or nginx) on the same device.

## Learn more

* Home page: [gemini://qwertqwefsday.eu/agate.gmi][home]
* [Cargo package][crates.io]
* [Source code][source]

## Installation and Setup

1. Get a binary for agate. You can use any of the below ways:

### Pre-compiled

Download and unpack the [pre-compiled binary](https://github.com/mbrubeck/agate/releases).

### NixOS/Nix

Using the nix package manager run `nix-env -i agate`

_Note:_ agate is currently only in the unstable channel and will reach a release channel once the next release is tagged

### Cargo

If you have the Rust toolchain installed, run `cargo install agate` to install agate from crates.io.

### Source

Download the source code and run `cargo build --release` inside the source repository, then find the binary at `target/release/agate`.

***
You can use the install script in the `tools` directory for the remaining steps if there is one for your system.  
If there is none, please consider contributing one to make it easier for less tech-savvy users!
***

2. Generate a self-signed TLS certificate and private key in the `.certificates` directory.  For example, if you have OpenSSL 1.1 installed, you can use a command like the following.  (Replace the *two* occurences of `example.com` in the last line with the domain of your Gemini server.)

```
mkdir -p .certificates

openssl req -x509 -newkey rsa:4096 -nodes -days 3650 \
    -keyout .certificates/key.rsa -out .certificates/cert.pem \
    -subj "/CN=example.com" -addext "subjectAltName = DNS:example.com"
```

3. Run the server. You can use the following arguments to specify the locations of the content directory, certificate and key files, IP address and port to listen on, host name to expect in request URLs, and default language code(s) to include in the MIME type for for text/gemini files: (Again replace the hostname `example.com` with the address of your Gemini server.)

```
agate --content path/to/content/ \
      --addr [::]:1965 \
      --addr 0.0.0.0:1965 \
      --hostname example.com \
      --lang en-US
```

All of the command-line arguments are optional.  Run `agate --help` to see the default values used when arguments are omitted.

When a client requests the URL `gemini://example.com/foo/bar`, Agate will respond with the file at `path/to/content/foo/bar`. If any segment of the requested path starts with a dot, agate will respond with a status code 52, whether the file exists or not. This behaviour can be disabled with `--serve-secret` or by an entry for the specific file in the `.meta` configuration file (see Meta-Presets). If there is a directory at that path, Agate will look for a file named `index.gmi` inside that directory.

## Configuration

### TLS versions

Agate by default supports TLSv1.2 and TLSv1.3. You can disable support for TLSv1.2 by using the flag `--only-tls13` (or its short version `-3`). This is *NOT RECOMMENDED* as it may break compatibility with some clients. The Gemini specification requires compatibility with TLSv1.2 "for now" because not all platforms have good support for TLSv1.3 (cf. §4.1 of the specification).

### Directory listing

You can enable a basic directory listing for a directory by putting a file called `.directory-listing-ok` in that directory. This does not have an effect on sub-directories.
The directory listing will hide files and directories whose name starts with a dot (e.g. the `.directory-listing-ok` file itself or also the `.meta` configuration file).

A file called `index.gmi` will always take precedence over a directory listing.

### Meta-Presets

You can put a file called `.meta` in any content directory. This file stores some metadata about the adjacent files which Agate will use when serving these files. The `.meta` file must be UTF-8 encoded.
You can also enable a central configuration file with the `-C` flag (or the long version `--central-conf`). In this case Agate will always look for the `.meta` configuration file in the content root directory and will ignore `.meta` files in other directories.

The `.meta` file has the following format (*1):
* Empty lines are ignored.
* Everything behind a `#` on the same line is a comment and will be ignored.
* All other lines must have the form `<path>:<metadata>`, i.e. start with a file path, followed by a colon and then the metadata.

`<path>` is a case sensitive file path, which may or may not exist on disk. If <path> leads to a directory, it is ignored.
If central configuration file mode is not used, using a path that is not a file in the current directory is undefined behaviour (for example `../index.gmi` would be undefined behaviour).
You can use Unix style patterns in existing paths. For example `content/*` will match any file within `content`, and `content/**` will additionally match any files in subdirectories of `content`.
However, the `*` and `**` globs on their own will by default not match files or directories that start with a dot because of their special meaning.
This behaviour can be disabled with `--serve-secret` or by explicitly matching files starting with a dot with e.g. `content/.*` or `content/**/.*` respectively.
For more information on the patterns you can use, please see the [documentation of `glob::Pattern`](https://docs.rs/glob/0.3.0/glob/struct.Pattern.html).
Rules can overwrite other rules, so if a file is matched by multiple rules, the last one applies.

`<metadata>` can take one of four possible forms:
1. empty  
    Agate will not send a default language parameter, even if it was specified on the command line.
2. starting with a semicolon followed by MIME parameters  
    Agate will append the specified string onto the MIME type, if the file is found.
3. starting with a gemini status code (i.e. a digit 1-6 inclusive followed by another digit) and a space  
    Agate will send the metadata whether the file exists or not. The file will not be sent or accessed.
4. a MIME type, may include parameters  
    Agate will use this MIME type instead of what it would guess, if the file is found.
    The default language parameter will not be used, even if it was specified on the command line.

If a line violates the format or looks like case 3, but is incorrect, it might be ignored. You should check your logs. Please know that this configuration file is first read when a file from the respective directory is accessed. So no log messages after startup does not mean the `.meta` file is okay.

Such a configuration file might look like this:
```
# This line will be ignored.
**/*.de.gmi: ;lang=de
nl/**/*.gmi: ;lang=nl
index.gmi: ;lang=en-UK
LICENSE: text/plain;charset=UTF-8
gone.gmi: 52 This file is no longer here, sorry.
```

If this is the `.meta` file in the content root directory and the `-C` flag is used, this will result in the following response headers:
* `/` or `/index.gmi`
    -> `20 text/gemini;lang=en-UK`
* `/LICENSE`
    -> `20 text/plain;charset=UTF-8`
* `/gone.gmi`
    -> `52 This file is no longer here, sorry.`
* any non-hidden file ending in `.de.gmi` (including in non-hidden subdirectories)
    -> `20 text/gemini;lang=de`
* any non-hidden file in the `nl` directory ending in `.gmi` (including in non-hidden subdirectories)
    -> `20 text/gemini;lang=nl`

(*1) In theory the syntax is that of a typical INI-like file and also allows for sections with `[section]` (the default section is set to `mime` in the parser), since all other sections are disregarded, this does not make a difference. This also means that you can in theory also use `=` instead of `:`. For even more information, you can visit the [documentation of `configparser`](https://docs.rs/configparser/2.0).

### Logging Verbosity

Agate uses the `env_logger` crate and allows you to set the logging verbosity by setting the default `RUST_LOG` environment variable. For more information, please see the [documentation of `env_logger`].

### Virtual Hosts

Agate has basic support for virtual hosts. If you specify multiple `--hostname`s, Agate will look in a directory with the respective hostname within the content root directory.
For example if one of the hostnames is `example.com`, and the content root directory is set to the default `./content`, and `gemini://example.com/file.gmi` is requested, then Agate will look for `./content/example.com/file.gmi`. This behaviour is only enabled if multiple `--hostname`s are specified.
Agate does not support different certificates for different hostnames, you will have to use a single certificate for all domains (multi domain certificate).

If you want to serve the same content for multiple domains, you can instead disable the hostname check by not specifying `--hostname`. In this case Agate will disregard a request's hostname apart from checking that there is one.

### Multiple certificates

Agate has support for using multiple certificates with the `--certs` option. Agate will thus always require that a client uses SNI, which should not be a problem since the Gemini specification also requires SNI to be used.

Certificates are by default stored in the `.certificates` directory. This is a hidden directory for the purpose that uncautious people may set the content root directory to the currrent director which may also contain the certificates directory. In this case, the certificates and private keys would still be hidden. The certificates are only loaded when Agate is started and are not reloaded while running. The certificates directory may directly contain a key and certificate pair, this is the default pair used if no other matching keys are present. The certificates directory may also contain subdirectories for specific domains, for example a folder for `example.org` and `portal.example.org`. Note that the subfolders for subdomains (like `portal.example.org`) should not be inside other subfolders but directly in the certificates directory. Agate will select the certificate/key pair whose name matches most closely. For example take the following directory structure:

```
.certificates
|-- cert.pem     (1)
|-- key.rsa      (1)
|-- example.org
|   |-- cert.pem (2)
|   `-- key.rsa  (2)
`-- portal.example.org
    |-- cert.pem (3)
    `-- key.rsa  (3)
```

This would be understood like this:
* The certificate/key pair (1) would be used for the entire domain tree (exceptions below).
* The certificate/key pair (2) would be used for the entire domain tree of `example.org`, so also including subdomains like `secret.example.org`. It overrides the pair (1) for this subtree (exceptions below).
* The certificate/key pair (3) would be used for the entire domain tree of `portal.example.org`, so also inclduding subdomains like `test.portal.example.org`. It overrides the pairs (1) and (2) for this subtree.

Using a directory named just `.` causes undefined behaviour as this would have the same meaning as the top level certificate/key pair (pair (1) in the example above).

The files for a certificate/key pair have to be named `cert.pem` and `key.rsa` respectively. The certificate has to be a X.509 certificate in a PEM file and has to include a subject alt name of the domain name. The private key has to be in PKCS#8 format. For an example of how to create such certificates see Installation and Setup, step 2.

## Logging

All requests will be logged using this format:
```
<local ip>:<local port> <remote ip or dash> "<request>" <response status> "<response meta>"[ error:<error>]
```
The "error:" part will only be logged if an error occurred. This should only be used for informative purposes as the status code should provide the information that an error occurred. If the error consisted in the connection not being established (e.g. because of TLS errors), the status code `00` will be used.

There are some lines apart from these that might occur in logs depending on the selected log level. For example the initial "Listening on..." line or information about listing a particular directory.

## Security considerations

If you want to run agate on a multi-user system, you should be aware that all certificate and key data is loaded into memory and stored there until the server stops. Since the memory is also not explicitly overwritten or zeroed after use, the sensitive data might stay in memory after the server has terminated.

[Gemini]: https://gemini.circumlunar.space/
[Rust]: https://www.rust-lang.org/
[home]: gemini://qwertqwefsday.eu/agate.gmi
[source]: https://github.com/mbrubeck/agate
[crates.io]: https://crates.io/crates/agate
[documentation of `env_logger`]: https://docs.rs/env_logger/0.8
