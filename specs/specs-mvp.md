# Project Specifications

This project should be a PWA that tracks my workouts. I should be able to install it on my Android like a
native app. It should have a backend REST API with a database and frontend that talks to that API.

Information in this document is just an idea, a draft. This means that it should be reviewed and better alternatives
should be considered and suggested. Terminology and component (classes, variables, functions, etc.) names are also
things that should be considered for improvements.

## Workouts

The project should support 2 different workouts represented by a single Workout entity. These 2 workouts are:

- Flow: Main Low Back Mobility Session
- Mobility Flow: Supporting Low Back Mobility Session

### Entities

- Workout
    - name: str
    - short_name: str
    - instructions: str
    - purpose: str (optional)
    - exercises: Exercise collection
- Exercise
    - workout: Workout (FK)
    - name: str
    - description: str
    - sets: int (optional)
    - reps: int (optional)
    - duration_sec: int (seconds) optional
    - load_kg: float (kilograms) optional
- WorkoutSession
    - workout: Workout
    - date: date
    - exercises: ExerciseSet collection
- ExerciseSet
    - exercise: Exercise
    - load_kg: float (kilograms) optional
    - duration_sec: int (seconds) optional

### Fixtures (seeds)

See [seeds.yaml](../seeds.yaml) for the full list of Workout and Exercise seed data (referenced by `workout` FK on
each Exercise).

## Architecture

### Backend

TODO

### Frontend

TODO
