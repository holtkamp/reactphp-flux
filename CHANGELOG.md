# Changelog

## 1.1.0 (2018-08-13)

*   Feature: Add `all()` helper to await successful fulfillment of all operations.
    (#11 by @clue)

    ```php
    // new: limit concurrency while awaiting all operations to complete
    $promise = Transformer::all($input, 3, function ($data) use ($browser, $url) {
        return $browser->post($url, [], json_encode($data));
    });

    $promise->then(function ($count) {
        echo 'All ' . $count . ' jobs successful!' . PHP_EOL;
    });
    ```

*   Feature: Forward compatibility with stable Stream v1.0 LTS.
    (#10 by @clue)

## 1.0.0 (2018-05-25)

*   First stable release, following SemVer

    I'd like to thank [@geertvanbommel](https://github.com/geertvanbommel),
    a fellow software architect specializing in database batch processing and
    API development, for sponsoring the first release! 🎉
    Thanks to sponsors like this, who understand the importance of open source
    development, I can justify spending time and focus on open source development
    instead of traditional paid work.

    > Did you know that I offer custom development services and issuing invoices for
      sponsorships of releases and for contributions? Contact me (@clue) for details.
