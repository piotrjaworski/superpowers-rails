---
name: rails-service-conventions
description: Use when creating or modifying service objects, extracting business logic from controllers or models, handling external API calls (Stripe, Transatel), or orchestrating multi-step workflows
---

# Rails Service Conventions

Services orchestrate multi-step workflows and external API calls. They are plain Ruby objects with a single responsibility.

## Core Principles

1. **One responsibility** - If naming feels awkward, split the service
2. **Single public method** - `call` only, exposed via `self.call`
3. **Return result objects** - Never raw `true`/`false`
4. **No exceptions for control flow** - Rescue and wrap in result, raise only for unexpected errors
5. **Never called from models** - Services sit above models, not beside them
6. **Thin jobs** - Sidekiq workers resolve IDs then delegate to a service

## Structure

```ruby
class UpgradeSubscription
  def initialize(account:, new_plan:)
    @account  = account
    @new_plan = new_plan
  end

  def call
    return failure("Plan not found") unless @new_plan.present?

    ActiveRecord::Base.transaction do
      upgrade_stripe_subscription
      update_sim_plan
    end

    success
  rescue Stripe::StripeError => e
    failure(e.message)
  end

  private

  attr_reader :account, :new_plan

  Result = Struct.new(:success?, :error, keyword_init: true) do
    def failure? = !success?
  end

  def success        = Result.new(success?: true, error: nil)
  def failure(error) = Result.new(success?: false, error: error)
end
```

## Result Objects
- Always return a `Result` struct with `success?`, `failure?`, `error`
- Rescue external errors inside the service and raise a custom one — never let the original error reach the controller
- Validate preconditions at the top of `call`, return `failure` early

## External APIs (Stripe, Transatel)
- Always rescue API-specific errors (`Stripe::StripeError`) inside the service
- Pass idempotency keys on all mutating calls, especially from Sidekiq
- Never call external APIs from models or controllers

## Persistence
- Wrap multi-model writes in `ActiveRecord::Base.transaction`
- Rescue `ActiveRecord::RecordInvalid` and return a failure result

## Sidekiq Integration
- Workers are thin: resolve IDs, call the service, handle result
- Raise on `result.failure?` to trigger Sidekiq native retry

# Namespace
- Use namespace to separate a business logic between different API integrations or business models/objects, for instance: `Stripe::UpgradeSubscription` or `Transatel::Api::ActiveSimCard`

## Quick Reference
| Do | Don't |
|----|-------|
| `UpgradeSubscription.call(...)` | Business logic in controllers |
| Return `Result` struct | Return `true`/`false` |
| Rescue API errors inside service | Let `StripeError` bubble up |
| Idempotency keys on Stripe mutations | Retry without idempotency |
| Thin Sidekiq workers | Fat workers with inline logic |
| Services call models | Models call services |

## Common Mistakes
1. **Fat workers** - Sidekiq jobs should only resolve IDs and call a service
2. **Returning booleans** - Callers need to know why it failed
3. **Calling services from models** - Services sit above models in the stack
4. **Missing rescue on external APIs** - Always wrap Stripe/Transatel calls
5. **Business logic in controllers** - If it's more than authorize + call + respond, extract it

**Remember:** Services orchestrate. Models own domain logic. Controllers authorize and delegate.
