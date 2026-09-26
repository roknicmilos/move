# Project Specifications

This project should be a PWA that tracks my physical activity. I should be able to install it on my Android like a
native app. It should have a backend REST API with a database and frontend that talks to that API.

Information in this document is just an idea, a draft. This means that it should be reviewed and better alternatives
should be considered and suggested.

## Workouts & Activities

The project should support multiple types of workouts / activities. Activities usually imply walking, hiking, swimming,
running/jogging and similar. These activities usually do not have any parts like workouts that have specific exercises.

### Activities

- Walking
    - started_at: datetime
    - ended_at: datetime
    - steps: int
- Running
    - pace: time (maybe int for seconds instead time?)
    - started_at: datetime
    - ended_at: datetime
    - distance_km: float
- Hiking
    - pace: time (maybe int for seconds instead time?)
    - started_at: datetime
    - ended_at: datetime
    - distance_km: float
    - lowest_elevation_m: int (meters)
    - highest_elevation_m: int (meters)

Make sure to crate a shared Activity class that holds shared fields and methods

### Exercises

- TODO: Workout
- TODO: ???

## Architecture

### Backend

TODO

### Frontend

TODO

## Additional Requirements

- Google Fit sync for daily steps
- Garmin sync for daily steps, running, hiking, etc.
- Walking activities should be automatically managed by the app. For example pull steps data from Google Fit and Garmin
  every hour or so.
