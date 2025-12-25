# frozen_string_literal: true

namespace :vendor do
  task :data do
    class RequirementsScrubber
      class << self
        def call(text)
          text.gsub!(/\[\[[fF]ile.*?\]\]/, "")

          text.gsub!(/\[\[(.*?|)?([^|]*?)\]\]/, '[[\2]]')

          text.gsub!(/<\/?span.*?>/, "")

          text.squeeze!(" ")

          text
        end
      end
    end

    class PrereqParser
      DEFEAT_AS_REGEX = /^Defeat \[\[.*?\]\] as \[\[(.*?)\]\]/
      OTHER_AS_REGEX = /\[\[(Home|Boss Rush|Completion Mark)\]\]s? as \[\[(.*?)\]\]/

      PREREQ_MAP = {
        "Isaac" => nil
      }

      class << self
        def call(text)
          prereqs = case text
          when DEFEAT_AS_REGEX
            [$1]
          when OTHER_AS_REGEX
            [$2]
          else
            []
          end

          prereqs.filter_map { |p| PREREQ_MAP.fetch(p) { p } }
        end
      end
    end

    require "net/http"
    require "json"
    require "uri"

    achievements = []

    base_uri = "https://bindingofisaacrebirth.wiki.gg/api.php?action=cargoquery&tables=achievement&format=json&fields=id,name,requirements&limit=500"

    [URI.parse(base_uri), URI.parse("#{base_uri}&offset=500")].each do |uri|
      response = Net::HTTP.get_response(uri)

      case response
      when Net::HTTPSuccess
        cargoquery = JSON.parse(response.body)

        $stdout.write "."

        achievements.concat(cargoquery["cargoquery"].map { |q| q["title"] })
      else
        $stdout.write "F"
      end
    end

    $stdout.puts

    achievements.each do |a|
      a["requirements"] = RequirementsScrubber.call(a["requirements"])
      a["prereqs"] = PrereqParser.call(a["requirements"])

      a["requirements"].gsub!(/\[\[|\]\]/, "")
    end

    JSON.dump(achievements, File.open("achievements.json.tmp", "w"))

    sh "jq . achievements.json.tmp > achievements.json"

    rm "achievements.json.tmp"
  end
end
