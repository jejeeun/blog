# frozen_string_literal: true

source "https://rubygems.org"

gemspec

gem "html-proofer", "~> 5.0", group: :test

# 로컬 전용 글 편집 UI — `bundle exec jekyll serve` 후 /admin/ 접속
# 배포 시엔 자동 제외(JEKYLL_ENV=production일 때 비활성)
group :jekyll_plugins do
  gem "jekyll-admin"
end

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.2.0", :platforms => [:mingw, :x64_mingw, :mswin]
