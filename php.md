# PHP Coding Standards

## Philosophy
  - Keep it simple, stupid. ( KISS )
  - Don't Repeat Yourself ( DRY )
  - Single Responsibility Principe ( SRP )

To help following these motto, we enforce the following rules from the [Object Calisthenics](https://github.com/TheLadders/object-calisthenics)
  - One level of indentation per method
  - Do not use the ``else`` keyword
  - Do not abbreviate

Other rules are encouraged but not forced upon for simplicity.
  
## Style and Indentation
Style and indentation should follow the agreed upon PSR's:
 - [PSR-1](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-1-basic-coding-standard.md)
 - [PSR-2](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md)

### Tools:
 - Sublime Text 3
 - Install phpcs `composer global require "squizlabs/php_codesniffer=*"` and then `sudo ln -s ~/.composer/vendor/bin/phpcs /usr/bin/phpcs`
 - Install php-cs-fixer `composer global require fabpot/php-cs-fixer @stable` and then `sudo ln -s ~/.composer/vendor/bin/php-cs-fixer /usr/bin/php-cs-fixer`
 - Open the Package Control, install "Phpcs"
 - Open its Preference > Package Settings > PHP Code Sniffer > user -settings and set this:
```javascript
{
    "php_cs_fixer_on_save": true,
    "php_cs_fixer_executable_path": "php-cs-fixer",
    "phpcs_executable_path": "phpcs",
}
```

### In HTML
 - Use ``<?= $myVar ?>`` to echo a variable.
 - Do not use curly brakets. Use the [Alternative syntax for control structures](http://www.php.net/manual/en/control-structures.alternative-syntax.php)


## Number rounding
 - Taxes must be stored as decimal, not float and handled with 4 decimals.
 - Always round (taxes, subtotals and totals) to two decimals for display purposes. Use number_format() for pages and round() for API responses.



