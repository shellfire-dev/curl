# `curl`: functions module for [shellfire]

This module provides a simple framework for using `curl` with a [shellfire] application. It wraps up the common API needed for using REST over HTTP(S). In particular, it provides a secure way of passing passwords, OAuth credentials and other sensitive headers and URLs to curl without exposing them to `ps` or via environment variables. It also integrates with `.netrc` and `.curlrc` so site-specific proxy settings and credentials can be kept separate from source control (and from users, where necessary) used without constantly reinventing the wheel.

An example user is the [github api] module. It is useful when combined with the [urlencode] module (eg for URL Templates) and the [jsonwriter] and [jsonreader] modules. The latter provides a fully JSON compliant event (ie SAX like) module to parse any arbitary JSON graph in shell script.

Headers are set to sensible defaults wherever possible, but can be overridden as necessary.

Settings are automatically adjusted when older versions of `curl` are detected (eg on CentOS 6.5).

## Compatibility

* Tag [`release_2015.0117.1750-1`](https://github.com/shellfire-dev/curl/releases/tag/release_2015.0117.1750-1) is compatible with [shellfire] release [`release_2015.0117.1750-1`](https://github.com/shellfire-dev/shellfire/releases/tag/release_2015.0117.1750-1).

## Overview

The main function to use is `curl_http()`. For example, to google's home page to standard out (`-`) with no user:-

```bash
local curl_httpVersion
local curl_httpStatusCode
local curl_httpDescription
curl_http none '' GET 'https://google.com/' '-'

my_callback()
{
	echo "Header field name: $fieldName"
	echo "Header field value: $fieldValue"
}

local headerFieldCount=0
my_other_callback()
{
	headerFieldCount=$((headerFieldCount+1))
}

curl_iterateHeaders my_callback my_other_callback

echo "There were $headerFieldCount headers"

```

## Importing

To import this module, add a git submodule to your repository. From the root of your git repository in the terminal, type:-
```bash
mkdir -p lib/shellfire
cd lib/shellfire
git submodule add "https://github.com/shellfire-dev/curl.git"
cd -
git submodule init --update
```

You may need to change the url `https://github.com/shellfire-dev/curl.git` above if using a fork.

You will also need to add paths - include the module [paths.d].

You will also need to import the [version] module.


## Namespace `curl`

### To use in code

If calling from another [shellfire] module, add to your shell code the line
```bash
core_usesIn curl
```
in the global scope (ie outside of any functions). A good convention is to put it above any function that depends on functions in this module. If using it directly in a program, put this line _inside_ the `_program()` function:-

```bash
_program()
{
	core_usesIn curl
	…
}
```

### Functions

***
#### `curl_http`

|Parameter|Value|Optional|
|---------|-----|--------|
|`authentication`|Authentication method to use|_No_|
|`credentials`|Use the function called `configure_validate_${configurationValidationFunction}` to validate configuration. Passed securely to `curl`|_No_|
|`verb`|HTTP verb to use.|_No_|
|`url`|URL to use, including query string. If needed, can be constructed from a url template using the [urlencode] module.|_No_|
|`outputFilePath`|File to write received body to. Use `-` for standard out (not advised, as you'd need to pipeline this call and so can't handle errors) or `/dev/null` if it's never wanted (again, not advised, as output might be an error moessage).|_No_|
|`…`|Key-Value pairs for headers, eg `Content-Type text/plain;charset=utf-8 Content-Language en`|_Yes_|

The functions sets the following variables on exit; normally you should set these to `local` variables before calling `curl_http()`.

|Parameter|Value|
|---------|-----|
|`curl_parsedHeadersFilePath`|Use this with the function `curl_iterateHeaders()`|
|`curl_httpVersion`|The HTTP version string (eg `HTTP/1.1`) from the final set of headers received|
|`curl_httpStatus`|The HTTP status code (eg `200`) from the final set of headers received|
|`curl_httpReasonPhrase`|The HTTP description (eg `OK`) from the final set of headers received|


The values of `authentication` may be:-

|Value|Description|
|-----|-----------|
|`none`|No authentication. `credentials` must be empty (`''`)|
|`basic`|HTTP Basic authentication. `credentials` can be user (`some_user`) or user and password (`some_user:some_password`)|
|`digest`|HTTP Digest authentication `credentials` can be user (`some_user`) or user and password (`some_user:some_password`)|
|`ntlm`|Windows NTLM authentication, if `curl` supports it|
|`kerberos`|Windows Kerberos authentication, if `curl` supports it|

The values of `verb` may be:-

|Value|Description|
|-----|-----------|
|`HEAD`||
|`GET`||
|`DELETE`||
|`POST`|Set `curl_uploadFile` to the file for the body to send. Contents must already be encoded. Given that web forms are rare for REST APIs, you might want to use the [jsonwriter] module to create JSON.|
|`PATCH`|Set `curl_uploadFile` to the file for the body to send. Contents must already be encoded. Given that web forms are rare for REST APIs, you might want to use the [jsonwriter] module to create JSON.|
|`PUT`|Set `curl_uploadFile` to the file for the body to send. Contents must already be encoded. Given that web forms are rare for REST APIs, you might want to use the [jsonwriter] module to create JSON.|

In addition, various out-of-bound parameters may be used to modify behaviour be set be specifiying variables `local` before calling `curl_http:_

|Out-of-band Parameter|Value|
|---------------------|-----|
|`curl_uploadFile`|Set this to a file to `POST`, `PATCH` or `PUT`. No form encoding is done (ie we use `--data-binary` to `curl`)|
|`curl_range`|Set this to values supported by `--range`|
|`curl_continueAt`|Set this to values supported by `--continue-at`|

Also, there are several global settings that can be overridden per-call (by using `local` variables, as above) or by configuration (see below). These are initially set by the internal `_curl_initialise()` function, which is called on first-use by `curl_http()`. 

|Global Setting|Value|Default|
|--------------|-----|-------|
|`curl_userAgent`|Value passed as User-Agent header|`shellfire`|
|`curl_maximumRedirects`|Maximum redirects|5|
|`curl_retries`|Maximum retries|10|
|`curl_retryDelay`|Delay in seconds between retries|0|
|`curl_retryMaximumDelay`|Maximum delay in seconds between retries|0|
|`curl_failHard`|Boolean. If true, then a non-zero curl exit code causes the application to exit|false|
|`curl_makeTlsInsecure`\*|Boolean|false for curl versions 7.19.7 and newer. true for older versions.|

Where necessary, older versions of `curl` are detected and settings that won't work are not used.

\* This effectively disables SSL/TLS certificate validation to workaround curl bugs with subjectAltName validation.

##### Using `.netrc`

The function will search for a `.netrc` file (useful for specifying passwords outside of the program) in the following locations\*:-

* `/etc/shellfire/netrc`†
* `/etc/${_program_name}/netrc`†
* `${CURL_HOME}/.netrc`
* `${HOME}/.netrc`
* `${HOME}/shellfire.netrc`
* `${HOME}/${_program_name}.netrc`

\* Not done on CentOS 6.5 and CentOS 5 because they use a very old version of `curl`. Instead only `$HOME/.netrc` is used if present.
† Actually, whatever path has been set inside the program for `/etc` - such as `/usr/local/etc`.

##### Using `.curlrc`

`.curlrc` lets you override most of the configuration choices made by `curl_http()` (but not those that affect URL use, file upload, etc). It is an ideal place to put site proxy credentials. The function will search for a `.curlrc` file in the the following locations:-

* `/etc/shellfire/curlrc`\*
* `/etc/${_program_name}/curlrc`\*
* `${CURL_HOME}/.curlrc`
* `${HOME}/.curlrc`
* `${HOME}/shellfire.curlrc`
* `${HOME}/${_program_name}.curlrc`

\* Actually, whatever path has been set inside the program for `/etc` - such as `/usr/local/etc`.

##### Tips

When expecting a reply, or even if there's the possibility of a reply, you'll want to keep `outputFilePath`. A good idea is to use a temporary file. Likewise, when using `POST`, `PUT` or `PATCH` repeatedly, it might be a good idea to create a temporary file and reuse it. For example:-

```bash
local TMP_FILE
core_temporaryFiles_newFileToRemoveOnExit
curl_uploadFile="$TMP_FILE"
…
jsonwriter_object \
	string name "$tagName" \
	string label "$commitish" \
	>"$curl_uploadFile"
```

If you do this, remember to truncate the file afterwards, eg:-

```bash
printf '' >"$curl_uploadFile"
```

***
#### `curl_iterateHeaders`

|Parameter|Value|Optional|
|---------|-----|--------|
|`…`|Zero or more callback functions (function names)|_Yes_|

Iterates over header fields after a call to `curl_http()`, calling each callback for each header field. Each callback will be able to look at the value of `fieldName` and `fieldValue`. Folded header fields are supported. `fieldValue` will have had leading and trailing whitespace removed; interior whitespace, however, isn't normalised. Useful for extracting headers such as `Location` or `Link`.



[swaddle]: https://github.com/raphaelcohn/swaddle "Swaddle homepage"
[shellfire]: https://github.com/shellfire-dev "shellfire homepage"
[core]: https://github.com/shellfire-dev/core "shellfire core module homepage"
[paths.d]: https://github.com/shellfire-dev/paths.d "paths.d shellfire module homepage"
[github api]: https://github.com/shellfire-dev/github "github shellfire module homepage"
[jsonwriter]: https://github.com/shellfire-dev/jsonwriter "jsonwriter shellfire module homepage"
[jsonreader]: https://github.com/shellfire-dev/jsonreader "jsonreader shellfire module homepage"
[urlencode]: https://github.com/shellfire-dev/urlencode "urlencode shellfire module homepage"
[unicode]: https://github.com/shellfire-dev/unicode "unicode shellfire module homepage"
[version]: https://github.com/shellfire-dev/version "version shellfire module homepage"
