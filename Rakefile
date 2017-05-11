# frozen_string_literal: true

require 'json'
require 'logger'

AWS_CLI = 'aws' # https://aws.amazon.com/cli/
LOGGER  = Logger.new($stderr)


namespace :ec2 do

  desc "get ID of instance with given value of tag Name"
  task :name_to_id, %i{name} do |*, name:|

    output AWS::EC2::Instance.by_name(name).id
  end

  desc "describe instance with given ID"
  task :get, %i{id} do |*, id:|

    output AWS::EC2::Instance.by_id(id).description
  end

  desc "force-stop and start instance"
  task :force_restart, %i{id} do |*, id:|

    instance  = AWS::EC2::Instance.by_id(id)
    log_state = ->(instance) { LOGGER.info "state: #{instance.state}" }
    log_state.call instance
    instance.stop force: true
    stopped       = false
    poll_interval = 21
    loop do
      sleep poll_interval
      instance = instance.reload
      log_state.call instance
      if !stopped && instance.stopped?
        instance.start
        stopped       = true
        poll_interval *= 2
      elsif stopped && instance.running?
        break
      end
    end
  end
end


module AWS

  def self.singular(result)
    if result.kind_of? Array
      if 1 == result.size
        result[0]
      elsif 1 < result.size
        raise "Ambiguous result:\n#{result.inspect}"
      else
        raise "Empty result"
      end
    else
      raise "Unexpected result:\n#{result.inspect}"
    end
  end


  module EC2

    class Instance

      attr_accessor :description

      def initialize(description)
        @description = description
      end

      def self.by_name(name)
        name = name.to_s.strip
        raise ArgumentError if name.empty?
        result = run %(#{AWS_CLI} ec2 describe-instances --filters 'Name=tag:Name,Values=#{name}')
        new(AWS.singular(described_instances(result)))
      end

      def self.by_id(id)
        id = id.to_s.strip
        raise ArgumentError if id.empty?
        result = run %(#{AWS_CLI} ec2 describe-instances --filters 'Name=instance-id,Values=#{id}')
        new(AWS.singular(described_instances(result)))
      end

      def self.described_instances(result)
        JSON(result)['Reservations'].map { |o| o['Instances'] }.flatten
      end

      def id
        description['InstanceId']
      end

      def state
        description['State']['Name']
      end

      %w{pending running shutting-down terminated stopping stopped}.each do |value|
        method_name = "#{value.tr('-', '_')}?"
        class_eval <<-RUBY, __FILE__, __LINE__ + 1
          def #{method_name}            # def shutting_down?
            #{value.inspect} == state   #   "shutting-down" == state
          end                           # end
        RUBY
      end

      def stop(force: false)
        run %(#{AWS_CLI} ec2 stop-instances #{force ? '--force ' : nil}--instance-ids #{id.inspect}),
            confirm_if: true, log: true
      end

      def start
        run %(#{AWS_CLI} ec2 start-instances --instance-ids #{id.inspect}),
            log: true
      end

      def reload
        self.class.by_id(id)
      end
    end
  end
end


def run(command, confirm_if: false, log: false)
  confirm command if confirm_if
  out = `#{command}`
  LOGGER.debug out if log
  out
end

def confirm(action)
  $stderr.print "#{action}\nConfirm [y/n]: "
  raise unless 'y' == STDIN.gets.strip
end

def output(data)
  $stdout.puts JSON.pretty_generate(data)
end
