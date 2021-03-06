2domain
============
**Splits a URL into sub-domain, domain and the top-level domain.**

Since domains are handled differently across different countries and organizations, splitting a URL into sub-domain, domain and top-level-domain parts is not a simple regexp. **2domain** uses a [large list of known top-level domains](https://publicsuffix.org/list/public_suffix_list.dat) from publicsuffix.org to recognize different parts of the domain.

This module uses a [trie](https://en.wikipedia.org/wiki/Trie) data structure under the hood to ensure the smallest possible library size and the fastest lookup. The library is roughly 30KB minified and gzipped. Since publicsuffix.org is frequently updated, the data structure is built on `npm install` as a `postinstall` hook. If something goes wrong during that step, the library falls back to a prebuilt list that has been built at the time of publishing.

<br />

# Installation

```sh
npm install 2domain
```

# Updating Database
You might need to update the database from time to time. Sometimes database downloading might fail and you might need to download it manually

```sh
node node-modules/2domain/scripts/build-tries.js
```

# Updating Database Strategy
I suggest update the files from crontab regularly. This repo returns a 0 if build-tries was successfully tested by mocha. So you might crontab like `node <path>/scripts/build-tries.js && node <path>/scrpits/write-pre.js` This would download and store the files to pre/ folder if mocha tests were successfull. Why would you need a pre/ folder: imagine you get internet outage while executing build-tries.js then you might have invalid or broken data. build-tries.js automatically taking the content from pre/ in case of such an error. That is why it it important to keep pre/ up2date, to recover from emergency cases.

Probably you would need to restart your services if they run long term too, because the .json files are read from tries modules and kept in memory during lifetime of the service. You might want to keep restarting your long-running services regularly or build something to detect changes and restart

# Usage

```javascript
// long subdomains can be handled
expect(parseDomain("some.subdomain.example.co.uk")).to.eql({
    subdomain: "some.subdomain",
    domain: "example",
    tld: "co.uk"
});

// protocols, usernames, passwords, ports, paths, queries and hashes are disregarded
expect(parseDomain("https://user:password@example.co.uk:8080/some/path?and&query#hash")).to.eql({
    subdomain: "",
    domain: "example",
    tld: "co.uk"
});

// unknown top-level domains are ignored
expect(parseDomain("unknown.tld.kk")).to.equal(null);

// invalid urls are also ignored
expect(parseDomain("invalid url")).to.equal(null);
expect(parseDomain({})).to.equal(null);
```

### Introducing custom tlds

```javascript
// custom top-level domains can optionally be specified
expect(parseDomain("mymachine.local",{ customTlds: ["local"] })).to.eql({
    subdomain: "",
    domain: "mymachine",
    tld: "local"
});

// custom regexps can optionally be specified (instead of customTlds)
expect(parseDomain("localhost",{ customTlds:/localhost|\.local/ })).to.eql({
    subdomain: "",
    domain: "",
    tld: "localhost"
});
```

It can sometimes be helpful to apply the customTlds argument using a helper function

```javascript
function parseLocalDomains(url) {
    return parseDomain(url, {
        customTlds: /localhost|\.local/
    });
}

expect(parseLocalDomains("localhost")).to.eql({
    subdomain: "",
    domain: "",
    tld: "localhost"
});
expect(parseLocalDomains("mymachine.local")).to.eql({
    subdomain: "",
    domain: "mymachine",
    tld: "local"
});
```

<br />

API
------------------------------------------------------------------------

### `parseDomain(url: string, options: ParseOptions): ParsedDomain|null`

Returns `null` if `url` has an unknown tld or if it's not a valid url.

#### `ParseOptions`

```javascript
{
    // A list of custom tlds that are first matched against the url.
    // Useful if you also need to split internal URLs like localhost.
    customTlds: RegExp|Array<string>,
    
    // There are lot of private domains that act like top-level domains,
    // like blogspot.com, googleapis.com or s3.amazonaws.com.
    // By default, these domains would be split into:
    // { subdomain: ..., domain: "blogspot", tld: "com" }
    // When this flag is set to true, the domain will be split into
    // { subdomain: ..., domain: ..., tld: "blogspot.com" }
    // See also https://github.com/peerigon/parse-domain/issues/4
    privateTlds: boolean - default: false
}
```

#### `ParsedDomain`

```javascript
{
    tld: string,
    domain: string,
    subdomain: string
}
```

<br />

## Tests

```sh
cd <project_path>
node_modules/mocha/bin/mocha --recursive -R dot node_modules/2domain/test/
```

### Forked
This repo was forked from [peerigon](https://github.com/peerigon/parse-domain) and modified/extended