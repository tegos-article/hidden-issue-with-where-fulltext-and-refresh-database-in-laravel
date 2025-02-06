# Hidden Issue with whereFulltext and RefreshDatabase in Laravel

## Overview

This article explores a hidden issue when using Laravel's `whereFulltext` method in tests that utilize the
`RefreshDatabase` trait. Since MySQL does not support FULLTEXT indexes within transactions, `RefreshDatabase` causes
tests relying on `whereFulltext` to return empty results. The article provides a detailed explanation of the issue and
presents practical solutions, including committing transactions manually and using dependency injection with a search
repository.

## Read the Full Article

[Hidden Issue with whereFulltext and RefreshDatabase in Laravel](https://dev.to/tegos/hidden-issue-with-wherefulltext-and-refreshdatabase-in-laravel-2p4f)

