# Keep up to date with `bundle update`.
# Install on new checkout with `bundle install`.
# Run with `bundle exec jekyll serve`.
source 'https://rubygems.org'
require 'json'
require 'open-uri'
versions = JSON.parse(open('https://pages.github.com/versions.json').read)

gem 'github-pages', versions['github-pages']
