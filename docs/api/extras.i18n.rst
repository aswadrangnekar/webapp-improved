.. _api.extras.i18n:

i18n
====
.. module:: webapp2_extras.i18n

This module provides internationalization and localization support for webapp2.
It requires the libraries `Babel <http://babel.edgewall.org/>`_ and
`pytz <http://pypi.python.org/pypi/pytz>`_ (or
`gaepytz <http://pypi.python.org/pypi/gaepytz>`_ for App Engine).

.. autodata:: default_config

.. autoclass:: I18nStore
   :members: translations, translations_path, domains,
             default_locale, default_timezone, date_formats, locale_selector,
             timezone_selector,
             __init__, set_locale_selector, set_timezone_selector,
             get_translations, load_translations

.. autoclass:: I18n
   :members: store, locale, translations, timezone, tzinfo,
             __init__, set_locale, set_timezone, gettext, ngettext,
             to_local_timezone, to_utc, format_date, format_datetime,
             format_time, format_timedelta, format_number, format_decimal,
             format_currency, format_percent, format_scientific, parse_date,
             parse_datetime, parse_time, parse_number, parse_decimal,
             get_timezone_location

.. autofunction:: get_store
.. autofunction:: set_store
.. autofunction:: get_i18n
.. autofunction:: set_i18n
.. autofunction:: lazy_gettext
.. autofunction:: gettext
.. autofunction:: ngettext
.. autofunction:: to_local_timezone
.. autofunction:: to_utc
.. autofunction:: format_date
.. autofunction:: format_datetime
.. autofunction:: format_time
.. autofunction:: format_timedelta
.. autofunction:: format_number
.. autofunction:: format_decimal
.. autofunction:: format_currency
.. autofunction:: format_percent
.. autofunction:: format_scientific
.. autofunction:: parse_date
.. autofunction:: parse_datetime
.. autofunction:: parse_time
.. autofunction:: parse_number
.. autofunction:: parse_decimal
.. autofunction:: get_timezone_location
