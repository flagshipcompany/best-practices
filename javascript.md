# Javascript Coding Standard

 - Styling guide: https://github.com/bevacqua/js/

## AngularJS Best Practice
 - Styling guide: https://github.com/johnpapa/angularjs-styleguide
   - use the Service style everywhere. Do not use factories.
 - ``ng-app`` should be in ``<body>``
 - The app should be the mother controller holding all submodules
 - When Injecting, place the Builtin Providers first, then ours.
 - Prefix all our modules with 'fcs.'
 - A function returning an AJAX promise should begin by `fetch` ( e.g. `fetchUser(123)` )
 
