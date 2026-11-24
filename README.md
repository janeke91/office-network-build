# AquaLeaf

**AquaLeaf** helps home greenhouse-and-pool owners automate climate, watering and pool safety - all in one mobile app.

## The problem

Our target user is a family or hobby grower with a small greenhouse that has a pool built into it. Today they:
- check pool pH and chlorine manually - easy to forget or get wrong
- deal with pool evaporation raising greenhouse humidity, which can damage plants
- find out about leaks, pump failures or temperature drops too late, after damage is done

## Our extra feature

AquaLeaf monitors and auto-corrects pool water quality and humidity balance, on top of the standard greenhouse automation - because our user's pool is part of the greenhouse, not a separate system.

**Direction (track):** Automation & Sustainability

## Features

| # | Module | How it works |
|---|--------|--------------|
| M1 | Login | Separate account per user; each sees only their own greenhouse |
| M2 | Dashboard | Air temperature, air humidity, soil moisture, pool pH - 4 live parameters |
| M3 | Control | Irrigation, ventilation, pool heater - 3 devices, on/off |
| M4 | Modes | Manual/auto switch; e.g. "soil moisture < 30% -> irrigation on", "pool pH out of range -> dosing on" |
| M5 | Alerts | Push alert on leak, pump failure, or sudden temperature change |
| M6 | History | Charts of temperature and humidity over day/week |
| M7 | Settings | User sets thresholds for watering, heating, and pool pH range |
| M8 | No internet | Clear "no connection" message, no crash, recovers automatically |

## Tech stack

- Flutter - one codebase for Android and iOS
- Firebase (Realtime Database/Firestore) - live data updates without extra backend code

## Team

| Member | Role |
|--------|------|
| Zhansaya (janeke91) | Backend / data layer (Firebase), security, integration |
| Saida | UI/UX design (Figma, style, logo) |
| Yeren | Mobile development (screens, navigation) |
| Aziza | Testing, documentation, presentation and demo |

## Project charter

Full charter (target user, extra feature, feature list, team setup): link here once shared

## Status

In progress.
