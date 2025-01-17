# Changelog

All notable changes to this project will be documented in this file.

Based on [Keep a Changelog](https://keepachangelog.com/).

This project adheres to [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [0.2.0] - 2017-06-25

### Added

- setuptools setup.py for distribution

### Changed

- account contact details are only updated if they change
- tell ACME server that all challenges are ready before verifying

### Fixed

- typos and text formatting issues

## [0.1.0] - 2017-06-19

### Added

- acme-hooked, based on [acme-tiny](https://github.com/diafygi/acme-tiny)
- hooks/template.sh - a template for bash hook scripts
- hooks/dns.sh - a customizable hook script for the DNS-01 ACME challenge
- hooks/http.sh - a customizable hook script for the HTTP-01 ACME challenge
- a wrapper and hook script that can act as a drop-in replacement for acme-tiny

### Changed

- can now process more than once CSR at once
- ACME registration now takes place with the contact address(es)
- when HTTP-01 challenges redirect to HTTPS, do not verify certificates

### Fixed

- do not error out when an ACME challenge is already completed
