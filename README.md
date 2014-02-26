gettexter - gettext for erlang
==============================

Goals of this project is:

* To be more or less compatible with GNU gettext (API and tools).
* To be as fast as possible in concurrent environment.

You may use this app to translate libraries as well as your own business apps
and use them in single installation with no conflicts.
Libraries may ship their own translations inside their distribution.

Quick Start
-----------

You definitely should be familiar with [GNU gettext](http://www.gnu.org/software/gettext/manual/gettext.html)
before start using this library.

In Erlang code include `shortcuts.hrl` header file.

```erlang
-include_lib("gettexter/include/shortcuts.hrl").
```

Mark your translatable strings with:

* `?_("")` - `gettext` (regular)
* `?N_("", "", N)` - `ngettext` (plural)
* `?P_(Ctx, "")` - `pgettext` (respecting msgctx)
* `?NP_(Ctx, "", "", N)` - `npgettext` (ngettext + pgettext)
* `?D_(Domain, "")` - `dgettext` (domain/namespace lookups, plus `?DN_`, `?DP_`, `?DNP_`)
* `?NO_("")` - gettext noop


```erlang
% file: my_app/src/my_module.erl
-module(my_module).
-define(GETTEXT_DOMAIN, my_app).
-include_lib("gettexter/include/shortcuts.hrl").

main(Name, What) ->
    gettexter:bindtextdomain(?GETTEXT_DOMAIN, "/.../locales"), % from where load locales
    gettexter:setlocale(undefined, "en"),  % locale for current process
    gettexter:textdomain(?GETTEXT_DOMAIN), % domain for current process

    Question = case What of
                 sleep -> ?NO_("Wanna sleep?");
                 eat -> ?NO_(<<"Wanna eat?">>)
               end,
    %% /* Translators: this is hello message */
    io:format(?_("Hello, ~p! ~ts"), [Name, ?_(Question)]).
```

Extract messages from sources to .pot file by `xgettext` command

```bash
export APP=my_app

xgettext -o locales/${APP}.pot --package-name=${APP} -d ${APP} --sort-by-file -L C \
     --keyword='NO_' --keyword='_' --keyword='N_:1,2' \
     --keyword='P_:1c,2' --keyword='NP_:1c,2,3' \
     --keyword='D_:2' --keyword='DN_:2,3' --keyword='DP_:2c,3' --keyword='DNP_:2c,3,4' \
     --add-comments=Translators \
     src/my_module.erl src/*.erl
```

Initialize new locale's .po file by `msginit`

```bash
mkdir -p locales/ru/LC_MESSAGES/
msginit -i locales/${APP}.pot -o locales/ru/LC_MESSAGES/${APP}.po --locale=ru
```

Or actualize existing locale's .po file by `msgmerge`

```bash
msgmerge -U locales/ru/LC_MESSAGES/${APP}.po locales/${APP}.pot
```

When translations are finished, generate locale's binary .mo files by `msgfmt`

```bash
msgfmt --check -o locales/ru/LC_MESSAGES/${APP}.mo locales/ru/LC_MESSAGES/${APP}.po
```
It's **strongly recommended** to not add .mo files to your repository (eg, add
`*.mo` to .gitignore / .hgignore) and generate them in compile-time (by rebar
post-compile hook or so).

API
---

Api tries to be compatible with [GNU gettext API](http://www.gnu.org/software/gettext/manual/gettext.html#gettext).

### Gettext lookups

```erlang
gettexter:gettext(string()) -> string().  % '?_' macro
```
Main gettext call. Uses locale, activated by `setlocale/2`.

```erlang
gettexter:ngettext(Singular :: string(), Plural :: string(),
                   Count :: integer()) -> string().  % '?N_' macro
```
Plural gettext call.

```erlang
gettexter:pgettext(Context :: string(), string()) -> string().  % '?P_'
gettexter:pngettext(Context :: string(), string(), string(),
                    integer()) -> string().  % '?PN_'
```
Gettext calls with respect to `msgctx`

```erlang
gettexter:dgettext(Domain :: atom(), string()) -> string().  % '?D_'
% and other 'gettexter:d{n,p,pn}gettext'
```
Gettext calls, which will search in a specific domain (namespace).
(See `bindtextdomain`).

### Gettext configuration

```erlang
gettexter:bindtextdomain(Domain :: atom(), LocaleDir :: file:filename()) -> ok.
```
Setup directory from which .mo files will be loaded like
`${LocaleDir}/${Locale}/LC_MESSAGES/${Domain}.mo`.
Domain **MUST** be specified for library applications (see `dgettext`).

By default, `Domain` is `application:get_application()` and `LocaleDir` is
`filename:absname(filename:join(code:lib_dir(Domain), "locale"))`.
If `LocaleDir` is relative, absolute path will be calculated, unlike original
`gettext` does (relative to cwd), but relative to `code:lib_dir(Domain)`.

This function don't reload already loaded locales for `Domain`, so, should be called
before `setlocale` or `ensure_loaded` calls.

This function usualy called only once at application initialization/configuration phase.

```erlang
gettexter:setlocale(lc_messages, Locale :: string()) -> ok.
gettexter:getlocale(lc_messages) -> string() | undefined.
```
Get / Set default locale for **current process**. It also loads .mo files for
`Locale` from the `LocaleDir`s for each `Domain` (if not loaded yet).
1'st argumet currently has only one value - `lc_messages` atom.
*Note: `getlocale` is not standard GNU gettext function.*

```erlang
gettexter:textdomain() -> atom() | undefined.
gettexter:textdomain(Domain :: atom()) -> ok.
```
Get/Set default domain (namespace) for **current process**.
If you call it like `textdomain(my_app)`, then `gettext(Key)` calls will be
converted to `dgettext(my_app, Key)`.
XXX: it's highly preferable to use `dgettext` directly and don't use this
API function, if localized strings can be rendered in different processes.

```erlang
gettexter:bind_textdomain_codeset(Domain :: atom(), Charset :: string()) -> ok.
```
Set encoding, on which all `gettexter:*gettext` responses should be converted.
This add significant performance overhead, and require 'iconv' dependency, if
.po file's `Content-Type` and `Charset` isn't the same.

XXX: Note, that `gettexter:gettext(Key)` call will be finally converted to
`gettexter:dpgettext(undefined, undefined, Key)`, which will try to extract
`Domain` and `Locale` from current process dictionary.

XXX: When developing library, you may want to re-define `?_`, `?N_`, `?P_` macroses
to call `dgettext` and use your module's domain by default.
You can do this by following trick:

```erlang
-define(GETTEXT_DOMAIN, my_domain).
-include_lib("gettexter/include/shortcuts.hrl").
```
This will modify macroses (except `?_D*`) to use `d*gettext(my_domain, ...)` by default.

### Proprietary APIs

This apis has no GNU gettext equiualents, but may be useful in Erlang apps.

```erlang
gettexter:which_domains(Locale) -> [atom()].
```
Which domains are loaded from .mo files to gettext server for `Locale'.

```erlang
gettexter:which_locales(Domain) -> [string()].
```
Which locales are loaded from .mo files to gettext server for `Domain'.

```erlang
gettexter:ensure_loaded(TextDomain, lc_messages, Locale) ->
    {ok, already | file:filename()} | {error, term()}.
```
Ensure, that locale is loaded from .mo file to gettexter server. If locale
isn't loaded, all `gettext` lookups to it will return default value `Msgid`.
This function may be called at application start-up or configuration time,
once for each supported locale.
In case of error, all data for this combination of `Domain` and `Locale` will
be removed from gettexter server to avoid incomplete/broken data.

```erlang
gettexter:reset() -> ok.
```
Remove all gettext stuff, added by `setlocale` or `textdomain`, from process
dictionary (but not from locale data storage). May be used to reuse process
for the next client.

Glossary
--------

* Domain: namespace for translations. In practice, the name of .po/.mo file, which
          in most cases, named as your OTP application.
* Locale: not strictly speaking, just name of translation's language, like "en",
          "en_GB", "ru", "pt_BR", etc. Usualy Locale contains also rules of
          plural form calculation, date/time/number formatting, currency etc.
* LC_MESSAGES: locale category, which contains translated application's strings (in
          .mo/.po format).

TODO
----

### Binary keys

Expression, `?_(<<"...">>)` is not well handled by `xgettext`, so, variants:

* Provide smth like `-define(B_(Str), gettexter_bin:gettext(Str)).` (which
  return binary translation for string key and add all of the `B*_` to `xgettext` keys.

* Allow to define return value of `?*_` macroses in module-level by smth like
  ```erlang
  -define(GETTEXT_USE_BIN, true).
  -include_lib("gettexter/include/shortcuts.hrl").
  ```
  and if `true`, in macroses replace `gettexter:*` calls to smth like `gettexter_bin:*`.

* Don't use macroses for binary gettext lookups, but use direct `gettexter_bin:*`
  calls. Pros: don't need to add all of them to `xgettext`, since function names
  are the same for `gettexter:*` and `gettexter_bin:*`.

### Custom locale loaders

Maybe allow `bindtextdomain/2` 2'nd argument be a `{M, F, A}` or `fun M:F/N` to
allow locale loading customization?
