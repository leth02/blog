---
title: "How to write an architecture document"
date: 2024-10-13
---
How to write a high level architecture document for a project spanning multiple backend service and client teams. Imagine that we are designing a new audio live streaming experience for users to listen to live sports games.


# 1. Introduction
Introduce the product at a high level, its individual use case(s) and feature(s).

This document provides the overall tech plan for Project X including an overview of the overall design and architecture.  Due to the broad scope of the project, this document is structured by User Stories to focus on the most significant architectural decisions to bring context to the overall design.  This document will also highlight other significant architecture and design decisions including the new & modified components/services, key interactions between components/services and domain model and recommendations.  While this document defines the high-level approach, sub-workstreams will flesh out additional details.

The goal of this document is to align stakeholders on:
- The overall architectural approach
- Define the solution-wide tenets
- New components and services being introduced
- Modifications and Enhancements required by existing services
- Outline of the end-to-end flows for the most architecturally significant user stories
- Dependencies, Integration points and technical requirements of those interactions
- Open questions and research activities to bring them to resolution
- Alternatives evaluated for key architectural decisions


This document does not:
- Provide detailed requirements or dives deep into specifics on each of the technical workstreams identified.  It is expected that this document will provide the alignment that enables each workstream to flesh out the detailed requirements and designs to drive them to completion. 
- Provide a detailed treatment of the dependent service architectures (e.g. Blueshift, Muse, AMC, Tenzing etc)
- Provide a timeline or resourcing plan for the dependent teams.

# 2. Stakeholders
List all services/teams involved and their key leader/point-of-contact. (service, client, product)

# 3. Tenets
These tenets will provide the guidelines for the Engineering teams to make the right decisions when faced with strong alternatives/trade-offs.

# 4. Terminology

# 5. Architecture


# 6. Summary of Functional User Stories
To support the proposed customer experience for the project, here are a summary of the major functional uses cases and their impact.

## 6.1 Browse


## 6.2 Playback on mobile and web clients


## 6.3 Text search

## 6.4 Playback of audio on demand (AOD)

## 6.5 Personalization

## 6.6 Notifications

## 6.7 Metrics

# 7. Technical workstreams
The use cases above are meant to give a high level overview of the work that needs to be done. we need to dive deeper to understand how we can meet these requirements. Below are technical workstreams that affect one or more of the use cases mentioned above. The workstreams are meant to drive investigation, additional technical requirements, design and implementation independently. 


# FAQ





