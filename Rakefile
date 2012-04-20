require 'fileutils'

# Path, where all Scriptogram articles are stored.
POSTS_PATH = File.expand_path('~/downloads/.DropboxEntooru/Dropbox/Apps/scriptogram/posts')

namespace :blog do

  desc 'Update Scriptogram with new articles from Github repo'
  task :update do
    repo_articles    = get_articles('articles')
    dropbox_articles = get_articles(POSTS_PATH)
    new_articles     = repo_articles - dropbox_articles

    unless new_articles.empty?
      new_articles.each do |article|
        puts "\e[1;32mAdding\e[1m \e[0;33m#{article}\e[0m..."
        convert_to_web(article)
      end

      puts "\n\e[0;32mNew articles have been added!\e[0m"
    end
  end

  # Retrieve all articles in the given path.
  #
  # path - Place, where your articles are stored.
  #
  # Returns Array of articles in the directory.
  def get_articles(path)
    Dir["#{path}/*"].map { |article| File.basename(article, '.md') }
  end

  # Convert file to Scriptogram format.
  #
  # file - File to convert.
  #
  # Returns nothing.
  def convert_to_web(file)
    file.match(/^(\d+)(_.+)$/)
    post = "#{POSTS_PATH}/#{file}.md"
    post_bak = "#{post}.bak"

    FileUtils.cp("articles/#{file}/ru#{$2}.md", post)
    FileUtils.cp(post, post_bak)

    title = File.open(post_bak, &:readline).chomp

    scriptogram_header =<<-TEXT
---
Date: #{$1}
Title: #{title}
---

    TEXT

    File.open(post, 'w') do |file|
      file.puts scriptogram_header
      bak = File.open(post_bak).to_a[3..-1]
      bak.each { |line| file.puts line }
    end

    FileUtils.rm(post_bak)
  end

end
