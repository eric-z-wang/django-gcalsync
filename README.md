=======
Django GcalSync
=============

Sync application models with Google Calendar, so events created or edited in the Google Calendar interface are automatically created/updated in your application. Helpful for situations were your application is the canonical record of event data created from various clients, including GC.

Authentication
----- 

If you haven't already, go to the [Google App Console](https://code.google.com/apis/console) and create an application. Create an OAuth client id and client secret for your application, and select "Download as JSON" when exporting. Update value assigned to `CLIENT_SECRETS` in `make_credentials.py` so it points to the file you just downloaded from the Google App Console.

    $ python make_credentials.py

You'll use the file generated by this script to authenticate with Google during the sync process.


Setup
-----

This needs to run as a demon/background process, and the assumption is that you're using (or will use) [django-celery](https://github.com/celery/django-celery) to accomplish this. 

Then...

Add `django-gcalsync` to your settings.py. 
 
Add a `GCALSYNC_CREDENTIALS` property to your settings.py - this should be the full path to the credentials file you created in Authentication above.

Add a `GCALSYNC_APIKEY` property to your settings.py - this is the key generated in the Google App Console in Authentication.

Assuming you're using [South](http://south.aeracode.org/), migrate models `python manage.py migrate django-gcalsync`

Run celery

    python manage.py celerybeat --log-level=info
    python manage.py celeryd —log-level=info


Usage
-----

Create a consumers.py module in the app containing the model you'd like to sync. In that module you'll create a class responsible for transforming Google Calendar event data so it's usable by your model. Your class must have a `transform` method that accepts an event_data dictionary (the data from Google).

For Example:

    from gcalsync import register
    from gcalsync.transformation import BaseTransformer
    from models import SomeEventModel

    class EventTransformer(BaseTransformer):
        model = SomeEventModel

        def transform(self, event_data):
            if not self.validate(event_data):
                return False

            start_datetime = self.parse_datetime(event_data['start']['dateTime'])
            end_datetime = self.parse_datetime(event_data['end']['dateTime'])

            return {
                'title': event_data['summary'],
                'start_date': start_datetime.date(),
                'start_time': start_datetime.time(),
                'end_date': end_datetime.date(),
                'end_time': end_datetime.time(),
                'url': event_data['htmlLink'],
                'event_id': event_data['id']
            }

Then register your transformer and associate it with a calendar that you'd like to sync with. In this example, "primary" is the name of the Google Calendar.

    register("primary", [EventTransformer])
