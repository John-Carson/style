# 37signals Development Philosophy

---

## Ship, Validate, Refine

- Merge "prototype quality" code to validate with real usage before cleanup
- Features evolve through iterations (tenanting: 3 attempts before settling)
- Don't polish prematurely - real-world usage reveals what matters

## Fix Root Causes, Not Symptoms

**Bad**: Add retry logic for race conditions
**Good**: Use `enqueue_after_transaction_commit` to prevent the race

**Bad**: Work around CSRF issues on cached pages
**Good**: Don't HTTP cache pages with forms

## Vanilla Rails Over Abstractions

- Thin controllers calling rich domain models
- No service objects unless truly justified
- Direct ActiveRecord is fine: `@card.comments.create!(params)`
- When services exist, they're just POROs: `Signup.new(email:).create_identity`

## When to Extract

- Start in controller, extract when it gets messy
- Filter logic: controller → model concern → dedicated PORO
- Don't extract prematurely - wait for pain
- Rule of three: duplicate twice before abstracting

## Rails 7.1+ `params.expect`

Replace `params.require(:key).permit(...)` with `params.expect(key: [...])`:
- Returns 400 (Bad Request) instead of 500 for bad params
- Cleaner, more explicit syntax

```ruby
# Before
params.require(:user).permit(:name, :email)

# After
params.expect(user: [:name, :email])
```

## Caching Constraints Inform Architecture

Design caching early - it reveals architectural issues:
- Can't use `Current.user` in cached partials
- Solution: Push user-specific logic to Stimulus controllers reading from meta tags
- Leave FIXME comments when you discover caching conflicts

---

## Core Philosophy: Earn Your Abstractions

### Question Every Layer of Indirection

**Pattern**: Consistently challenge abstractions that don't justify their existence.

> "I find these explicit classes for the notifier rather anemic. And there's not as much future potential for a million more (unlike Basecamp). Think we're better off inlining them."

> "Good example of how this is getting confusing and very indirect between source and resource. And there just aren't enough variations to warrant this level of indirection."

**The Test**: Ask "Is this abstraction earning its keep?" If you can't point to 3+ variations that need it, inline it.

### "Anemic" Code Should Be Inlined

**Pattern**: Methods and classes that don't explain anything or provide meaningful abstraction should be removed.

> "Don't think this method is carrying its weight. Either it needs to explain something or you should just inline."

> "Bit anemic. Would inline."

> "Don't think this association definition should be necessary. We should be able to use everything from the delegated types."

**Rule**: If a method just wraps another call with no additional logic or explanation, delete it.

---

## Write-Time vs Read-Time Operations

### Compute at Write Time, Not Presentation Time

> "All this manipulation has to happen when you save, not when you present. So data model has to fit something where it can be updated. Otherwise you won't be able to paginate."

> "Don't think this approach is going to fly. These threads need to be paginated, so you can't do any in-memory sorting. This all needs to be converted to a delegated type, so you have a single table you can pull from."

> "Would consider storing the current summary as a body here. Then you only compute this when there's a write."

> "These are some awfully complicated queries. Would consider a way to compute the sort code at write time instead."

**Pattern**:
```ruby
# Bad - computing at read time
def thread_entries
  (comments + events).sort_by(&:created_at)
end

# Good - using delegated types with single-table query
class Message < ApplicationRecord
  delegated_type :messageable, types: %w[Comment EventSummary]
end

# Now you can paginate:
bubble.messages.order(:created_at).limit(20)
```

**Why it matters**:
- Enables pagination
- Enables caching
- Removes complexity from views

---

## Database Over Application Logic

### Prefer DB Constraints Over AR Validations

> "Don't think these validations add much/anything over just having the DB raise an exception if, say, uniqueness constraint is violated... Generally speaking, we've almost entirely stopped using validations like this."

> "Another validation that can just be a db constraint."

**Pattern**:
```ruby
# Avoided
class JoinCode < ApplicationRecord
  validates :code, uniqueness: true
  validates :usages, numericality: { greater_than_or_equal_to: 0 }
end

# Preferred
# In migration:
add_index :join_codes, :code, unique: true
# Let the database enforce integrity
```

**When to validate**: Only when you need user-facing error messages for form display.

### Use AR Counter Caches

> "Should use AR counters: https://api.rubyonrails.org/v7.1/classes/ActiveRecord/CounterCache/ClassMethods.html"

> "You can lean on the AR counter methods here for a more natural API."

---

## Naming Principles

### Use Positive Names

> "`not_popped` is pretty cumbersome of a word. Consider something like `unpopped` if staying in the negative or go with something like `active`. Probably better with the latter."

**Pattern**:
```ruby
# Avoid
scope :not_popped, -> { where(popped_at: nil) }
scope :not_deleted, -> { where(deleted_at: nil) }

# Prefer
scope :active, -> { where(popped_at: nil) }
scope :visible, -> { where(deleted_at: nil) }
```

### Method Names Should Reflect Their Return Value

> "`collect` implies that we're returning an array of mentions (as #collect). Would use `create_mentions` when you don't care about the return value."

### Consistent Domain Language

> "`container` strikes me as out of context with mentions. We don't use that term anywhere else. Isn't this the same as the `source` concept we refer to in Notifications?"

> "This is a bit confusing. You can't really tell the difference between source and resource. Could we get clearer about this?"

> "Should probably be `messages` now that you've dropped the `thread` domain name for the rest of the feature. And make that consistent throughout."

---

## Rails Conventions

### StringInquirer for Action Predicates

> "Bit too heavy-handed, imo. Better to make action return a StringInquirer. Then you can do `event.action.completed?`."

```ruby
# Instead of method_missing magic or case statements:
class Event < ApplicationRecord
  def action
    self[:action].inquiry
  end
end

# Usage:
event.action.completed?
event.action.published?
```

### Use `after_save_commit` Shorthand

> "You can use `after_save_commit` instead of `after_commit on: %i[ create update ]`."

### Prefer `pluck` Over `map`

> "Use `pluck(:name)` instead of `map(&)`."

> "Don't think you need this accessor if you just use pluck at the callsite, so it's `event.assignees.pluck(:name)`."

### Delegate for Lazy Loading

> "Why not just delegate :user to :session? Then you get to lazy load it too."

### Touch Chains for Cache Invalidation

> "Needs to `touch: true` to bust caching."

---

## View Patterns

### Extract View Logic to Helpers, Not Partials

> "Something about this feels slightly off. Maybe it's the fact that the partials are really more just like helper methods. There's virtually no html in them."

> "Smells like this should be a method on the EventSummary. There's no markup here. And there's feature envy."

**Pattern**: If a partial has virtually no HTML and is mostly Ruby logic, it should be:
1. A helper method (if view-specific)
2. A model method (if it's domain logic)

### Helpers Should Receive Explicit Parameters

> "Generally consider it a smell to have helpers refer to magical ivars. Better to pass in the ivar to make that dependency explicit."

```ruby
# Bad - relies on @bubble ivar
def bubble_activity_count
  @bubble.comments_count + @bubble.events_count
end

# Good - explicit dependency
def bubble_activity_count(bubble)
  bubble.comments_count + bubble.events_count
end
```

### Double-Indent Attributes in Tag Helpers

> "Fix indention by double-indenting the attributes to the yielding method."

```erb
<%# Bad %>
<%= tag.div class: "foo",
  data: { controller: "bar" } do %>

<%# Good %>
<%= tag.div class: "foo",
    data: { controller: "bar" } do %>
```

### Use Tag Helpers for Meta Tags

> "Would use a tag helper when you're doing interpolation like this."

```erb
<%# Instead of string interpolation: %>
<meta name="current-user-id" content="<%= Current.user.id %>">

<%# Use tag helper: %>
<%= tag.meta name: "current-user-id", content: Current.user.id if Current.user %>
```

### Turbo Stream Canonical Style

> "This should also use the canonical style: `turbo_stream.update [ @card, :new_comment ], partial: \"cards/comments/new\", locals: { card: @card }`"

> "Should use consistent style here, so `[ @card, :new_comment ]`, like we do in the destroy template."

---

## JavaScript / Stimulus Patterns

### Targets Over CSS Selectors

> "Yeah this one is a bit odd. These should just be targets rather than using a css selector."

### Consider WebSocket Updates in Controllers

> "Is this going to catch new elements added via web socket? Thinking it probably won't. Maybe you need to extract this and also call it when element is added."

---

## Migrations

### Migrations Can Reference Models

> "Full `db:migrate` is an antipattern in my book. Migrations were only ever meant to be transient. To get a schema from one version to the next. Interacting directly with models present at the time of the migration is totally fine."

```ruby
# This is fine in migrations:
Notification.update_all(source_type: "Event")

# Instead of raw SQL:
execute "UPDATE notifications SET source_type = 'Event'"
```

---

## Be Explicit Over Clever

### Avoid Introspection Magic

> "Actually, I think this is too clever. There are only two different types of cards that have mentionable content: cards and comments. I would find a way to be explicit about this. Like just letting Mentionables define the 'mentionable_content' method."

> "Good example here too where the method_missing actually works a bit against you. You're probably better off with a `case event.action`."

**Pattern**: When there are only 2-3 cases, explicit `case` statements or defined methods beat metaprogramming.

### Avoid Unnecessary Base Class Extensions

> "This is a bit too much. Should just put this method on the Reaction class. Should be very hesitant to add base class extensions, and we should only go there if it's on its way to an upstream Active Support patch."

---

## Caching Principles

### Avoid Complex Cache Dependencies

> "Really don't like all these dependency on the base cache either. Better to flip it around to use lazy loading or touches. This is essentially the same thing anyway."

> "I really don't like the idea that this base page is cache dependent on anything beyond itself."

**Pattern**: Use `touch: true` on associations rather than adding cache key dependencies that span multiple models.

### Use `update_all` for Bulk Updates

> "I'd be surprised if we need this? We should just be able to do a `cards.update_all`. There aren't any side effects we're hoping to run here. This is just to bump the caches."

---

## API Design

### Implicit Respond To

> "We don't need a respond_to block when the action has templates for both formats. That'll automatically be implied."

```ruby
# Unnecessary:
def show
  respond_to do |format|
    format.html
    format.json
  end
end

# Just have show.html.erb and show.json.jbuilder - Rails handles it
def show
end
```

### Use Inline Jbuilder Partials

> "Inline." / "Inline as well."

> "Can just use `json.steps @card.steps, partial: \"steps/step\", as: :step`."

```ruby
# Instead of explicit render calls:
json.steps do
  json.array! @card.steps do |step|
    json.partial! "steps/step", step: step
  end
end

# Use inline style:
json.steps @card.steps, partial: "steps/step", as: :step
```

### Prefer `head :no_content` for Updates

> "Why use the `render :show` here vs `head :no_content`?"

---

## Routing

### Use My:: Namespace for Current User Resources

> "This should be `My::IdentitiesController`. We're putting everything that derives from `Current.identity` on that to imply there won't be a /identities/x."

> "Should stick with the `My::AvatarsController` format we have for the rest of the `/my` namespace."

---

## Data Defaults

### Use `created_at` for Initial Timestamps

> "Do we need to take this or could we just set `last_active_at = created_at` on first creation?"

---

## Authorization Patterns

### Unauthenticated Implies Unauthorized

> "This smells a little. If we're allowing unauthenticated access, it should be implied that we're also allowing unauthorized access, since you can't authorize someone you haven't authenticated. Let's level up `allow_unauthorized_access` to be able to do this directly."

---

## Key Takeaways

1. **Abstractions must earn their keep** - If it doesn't explain or enable variations, inline it
2. **Write time > Read time** - Compute summaries and sort keys when saving, not presenting
3. **Database over AR** - Prefer DB constraints over ActiveRecord validations
4. **Positive names** - Use `active` not `not_deleted`
5. **Explicit over clever** - Case statements beat metaprogramming for 2-3 variations
6. **StringInquirer for predicates** - `action.completed?` over string comparisons
7. **Touch chains** - Use `touch: true` for cache invalidation
8. **Helpers take params** - Don't rely on magical ivars
9. **Targets over selectors** - In Stimulus, use data-*-target
10. **Tests shouldn't shape design** - Never add code just for testability

---

## UX-First Decision Making

### Perceived Performance > Technical Performance

> "I'd imagined this as a single form in the sense that you'd make all of your selections and then 'apply' the filter rather than it updating the view after every new choice. Some shopping websites do that latter and it always feels/is slow."

**The Pattern**:
- User perception matters more than server response time
- Even if technically fast locally, if it *feels* slow in real conditions, it needs rethinking
- Compare to familiar patterns users encounter elsewhere ("shopping websites")

**Transferable Lesson**:
When reviewing implementations, ask: "Does this feel instant to the user?" Not just "Is the server response under 200ms?"

### Simplify by Removing, Not Just Hiding

> "One thing we could try if it were to simplify things is to not show the chips while the form is open. Then there wouldn't be anything to update live on the page."

**The Pattern**:
- If real-time updates add complexity, question whether they're needed
- Reduce what's visible during interaction, not just what's rendered
- Simpler UI states = fewer edge cases = more reliable UX

**Example Application**:
```ruby
# Instead of live-updating summaries while editing
# Show simple form → Apply → Show updated summary
# Less JS, fewer round-trips, clearer mental model
```

---

## Prototype Quality Shipping

### Explicitly Label Implementation Quality

**The Pattern**:
- Communicate implementation quality expectations upfront
- "Prototype quality" is a valid shipping standard when validating
- Different features deserve different polish levels based on uncertainty

**Key Quote**:
> "The goal is to get this onto our production instance as soon as possible so we can vet the design with real work."

### Ship to Validate, But Document Known Issues

1. Performance issues: "un-holy things with Bubble collections"
2. Missing features: "no pagination in the new view"
3. Technical debt acknowledged: "mess we haven't cleaned up since we moved to the cards design"

**The Pattern**:
- Ship with known flaws if they don't block validation
- Enumerate specific areas needing attention
- Distinguish between "needs real data to know" vs "clearly broken"

**Transferable Application**:
```ruby
# In PR description template:
## Known Limitations (acceptable for validation)
- [ ] Performance not optimized (works locally, needs prod data)
- [ ] No pagination (start without to test core UX)
- [ ] Old partials not removed (cleanup after validation)

## Blockers (must fix before merge)
- [ ] Data loss potential
- [ ] Security issues
```

---

## Real Usage Trumps Speculation

### Prefer Production Validation Over Local Perfection

> "A concern is that it currently runs slowly on our beta instance which may be simply because the Digital Ocean droplet doesn't have sufficient specs. It's quite fast on local dev so this might not be an issue at all on our production instance."

**The Pattern**:
- Different environments tell different stories
- Production behavior with real data > development behavior with fixtures
- "Could be that you just merge it as is and there's no problem"

**Decision Framework**:
1. Is it fast enough locally? → Suggests architectural approach is sound
2. Is it slow in staging? → Could be infrastructure, not code
3. Ship to prod to know for sure (with monitoring ready)

### Name Technical Debt, Don't Block on It

> "There is also some mess here that we haven't cleaned up since we moved to the cards design... the whole `Bubble` namespace doesn't really make sense anymore but we haven't done anything about it. I'm only pointing that out because it's probably confusing!"

**The Pattern**:
- Acknowledge confusing code without demanding immediate fixes
- Context for reviewers: "this exists in main, too"
- Prevents blocking valuable features on cleanup

---

## Incremental Feature Addition

### Add Escape Hatches Without Removing Primary Path

The PR added a "Create and add another" button without changing the primary "Create card" flow.

**Code Pattern**:
```ruby
# Before: Single action
<%= button_to "Create card", card_publish_path(card) %>

# After: Primary + escape hatch
<%= button_to "Create card", card_publish_path(card),
    name: "creation_type", value: "add" %>
<%= button_to "Create and add another", card_publish_path(card),
    name: "creation_type", value: "add_another" %>
```

**The Pattern**:
- Don't replace existing behavior, extend it
- Use form parameters to branch: `params[:creation_type]`
- Keeps both paths working, no feature flags needed

**Controller Implementation**:
```ruby
def create
  @card.publish
  redirect_to add_another_param? ? @collection.cards.create! : @card
end

private
  def add_another_param?
    params[:creation_type] == "add_another"
  end
```

**Transferable Lesson**:
When adding workflows, branch with parameters rather than replacing routes or creating separate endpoints.

---

## Visual Polish Through Iteration

### Ship Visual Redesigns Big

This was a massive visual overhaul (95 files changed):
- Complete CSS restructuring
- New card-based layouts
- Pinning system
- New color schemes

**The Pattern**:
- Visual redesigns are better done wholesale than piecemeal
- Easier to evaluate coherence when everything updates together
- Screenshots in PR (not just code) for visual review

**Why This Matters**:
Incremental visual changes create inconsistent UX. Better to:
1. Design the new vision completely
2. Implement it all at once
3. Validate with real usage
4. Iterate on the new baseline

---

## Feature Design Principles

### Reuse Robust Systems for New Features

> "The whole thing runs through the `Filter` system. It's quite robust and resilient so I got a lot of mileage out of breaking it out into individual forms to get the various sorting and filtering in place."

**The Pattern**:
- Identify "robust and resilient" existing systems
- Leverage them creatively for new features
- Even if "there are certainly better ways" - ship with what works

**Example**:
```ruby
# Instead of building new sorting UI from scratch
# Use existing filter system with different params
<%= form_with url: bubbles_path, method: :get do %>
  <%= hidden_field_tag :indexed_by, "newest" %>
<% end %>

<%= form_with url: bubbles_path, method: :get do %>
  <%= hidden_field_tag :indexed_by, "oldest" %>
<% end %>
```

---

## Feedback Style

### Give Product Context, Not Implementation Mandates

> "I'd imagined this as a single form in the sense that you'd make all of your selections and then 'apply'..."

**The Pattern**:
- Share the product vision ("I'd imagined...")
- Explain the UX concern ("feels/is slow")
- Suggest an approach ("One thing we could try...")
- Let the implementer figure out how

**Not**:
- "Change this to use X technology"
- "The performance is unacceptable"
- "You must do Y"

### Trust, Then Verify in Production

> "It could be that you just merge it as is and there's no problem. If you do choose to dig more deeply, here are a few areas to look into..."

**The Pattern**:
- Default to shipping
- Provide investigation paths if reviewer wants to dig deeper
- "Factor your appetite accordingly" - let reviewer decide effort level

---

## CSS Container Query Patterns

### Use Container Queries for Responsive Cards

```css
.card {
  container-type: inline-size;
  font-size: 1.8cqi;  /* Container query units */
}

.card__title {
  font-size: 2.5em;  /* Relative to container */
}
```

**The Pattern**:
- Components size themselves based on container, not viewport
- `cqi` units (container query inline size)
- Cards work anywhere because they're self-contained

**Why This Matters**:
Cards can appear in:
- Grid layouts (3 columns)
- List layouts (single column)
- Sidebars (narrow)
- Modals (medium)

All without media queries.

---

## Data-Driven Development

### List Specific Investigation Areas

When shipping for validation, enumerate what to watch:

> "If you do choose to dig more deeply, here are a few areas to look into:
> 1. The `Bubbles#Index` view is doing a lot of un-holy things...
> 2. There's also no pagination...
> 3. The whole thing runs through the `Filter` system...
> 4. Look for cases that might cause data loss..."

**The Pattern**:
- Number specific areas of concern
- Point to exact code locations
- Note what's "un-holy" vs what's intentional-but-rough
- Call out data safety specifically

---

## Key Takeaways

1. **Feel > Metrics**: User perception beats benchmarks
2. **Ship to Learn**: "Prototype quality" is a valid standard for validation
3. **Context Over Criticism**: Explain the mess without blocking the feature
4. **Extend, Don't Replace**: Add new paths via parameters, keep old paths working
5. **Production Truth**: Real data reveals what local testing can't
6. **Leverage What Works**: Reuse robust systems even if not "perfect" for new use case
7. **Visual Coherence**: Ship visual redesigns wholesale, not piecemeal
8. **Enumerate Concerns**: List specific areas to investigate, not vague "needs work"

---

## Application to Your Projects

### PR Template Addition
```markdown
## Shipping Standard
- [ ] Production quality - polished and complete
- [ ] Prototype quality - validating approach, known limitations below
- [ ] Experimental - testing feasibility only

## If Prototype Quality
### What We're Validating
-

### Known Limitations (acceptable for validation)
-

### Will Investigate After Real Usage
-
```

### Code Review Questions
1. Does this feel fast to users? (not just: is it fast?)
2. Are we shipping to learn or shipping to finish?
3. What will real data tell us that fixtures can't?
4. Are we extending or replacing? (prefer extending)
5. If there's mess, is it documented? Is it blocking?

---

### Public vs Private Surface Area

**Pattern**: Aggressively minimize public methods.

```ruby
# Bad - exposes internal details
class Quota
  def reset_quota
  def check_if_due_for_reset
  def calculate_usage
end

# Good - narrow public API
class Quota
  def spend(cost)
  def ensure_not_depleted

  private
    def reset_if_due
    def depleted?
end
```

**Feedback**:
> "I'd use the rule of not adding public methods that are not used anywhere. The narrower the public surface of a class the better, since it's easier to grasp its responsibilities at a glance."

**Why it matters**:
- Easier to understand class responsibilities
- Signals internal vs external concerns
- Prevents coupling

---


### Domain-Driven Naming

**Pattern**: Choose names that reflect business reality, not implementation.

**Examples**:

```ruby
# Treat quota as something you "spend"
quota.spend(cost)           # not increment_usage(cost)
quota.ensure_not_depleted   # not ensure_under_limit
quota.depleted?             # not over_limit?

# Reasoning: "A quota is something you spend until you don't have anything left"
```

**Why it matters**: Code reads like the domain experts talk about it.

---

## Architecture Decisions

### Introduce Proper Objects When State Couples

**Pattern**: When parameters get passed through multiple method layers, extract an object.

**Before (code smell)**:
```ruby
def cost(within:)
  # ...
end

def cost_microcents(within:)
  # ...
end

def limit_cost(within:)
  # ...
end
```

**Feedback**:
> "The `limit` param results in having to pass it down the pipeline to several other methods. The shared param is often a smell that something is missing."

**After - Extracted `Ai::Quota` model**:
```ruby
class Ai::Quota < ApplicationRecord
  def spend(cost)
  def ensure_not_depleted
  def reset_if_due
end
```

**Why it matters**:
- Eliminates parameter coupling
- Encapsulates related behavior
- Creates clear ownership of concepts

---


### Custom Types: Only When Justified

**Pattern**: Consider custom Active Model types, but weigh the cost.

**Discussion** about `Money` type:

```ruby
# Could be done with custom type
class MoneyType < ActiveModel::Type::Value
  def cast(value)
    # handle "$100", 100, BigDecimal...
  end
end

attribute :limit, :money_type
attribute :used, :money_type
```

But then reconsidered:
> "The more I think about this, the more I feel that doing this conversion like this is totally fine. I feel like the whole custom type / accessor thing for money is too much."

**Final approach - Value object**:
```ruby
class Ai::Quota::Money < Data.define(:value)
  MICROCENTS_PER_DOLLAR = 100 * 1_000_000

  def self.wrap(value)
    case value
    when String then convert_dollars_to_microcents(BigDecimal(value[NUMBER_REGEX]))
    when Integer then new(value)
    # ...
    end
  end

  def in_microcents
    value
  end

  def in_dollars
    in_microcents.to_d / MICROCENTS_PER_DOLLAR
  end
end

# Usage
quota.spend(Money.wrap("$5.00"))
message.cost.in_dollars
```

**Why this approach won**:
- Money conversion only happens in one place (`Ai::Quota`)
- Value object encapsulates arithmetic
- Avoids framework overhead
- Fixed-point arithmetic (no float errors)

**Decision criteria**:
> "If we kept messing around with quota amounts in several other places in the app, then the custom type could be justified. But I think the current approach is simple enough."

---


### Concerns: Public Behavior Only

**Pattern**: Don't extract concerns containing only private methods.

```ruby
# Initial attempt - concern with reset logic
module Ai::Quota::Resettable
  private
    def reset_if_due
    def due_for_reset?
end

# Final - inlined into main class
class Ai::Quota
  def spend(cost)
    reset_if_due
    increment!(:used, cost.in_microcents)
  end

  private
    def reset_if_due
      reset if due_for_reset?
    end
end
```

**Rule**:
- **Concern**: Auxiliary public traits (Attachable, Named, Mentionable)
- **Private methods**: Inline in main class unless very large

---


### Wrapping Methods: Hide or Reveal?

**Pattern**: Consider whether wrapping methods hide useful details or just add noise.

Initial thought:
```ruby
# Wrapper seems noisy
user.ensure_ai_quota_not_depleted
user.spend_ai_quota(cost)

# Direct access cleaner?
user.ai_quota.ensure_not_depleted
user.ai_quota.spend(cost)
```

Then reconsidered:
> "Sorry, I notice now that the wrapping methods were dealing with the 'lazy creation' of AI quota. I think it's fine as it was 👍"

**Final pattern**:
```ruby
module User::AiQuota
  def spend_ai_quota(cost)
    fetch_or_create_ai_quota.spend(cost)
  end

  private
    def fetch_or_create_ai_quota
      ai_quota || create_ai_quota!(limit: DEFAULT_QUOTA)
    end
end
```

**Why it matters**: Lazy initialization is internal detail worth hiding.

---

## Performance Patterns

### Memoize in Hot Paths

**Pattern**: Cache method results that are called repeatedly during rendering.

```ruby
# Before - called many times during page render
def as_params
  {}.tap do |params|
    params[:indexed_by] = indexed_by
    params[:sorted_by] = sorted_by
    # ... many queries ...
  end
end

# After
def as_params
  @as_params ||= {}.tap do |params|
    params[:indexed_by] = indexed_by
    # ...
  end
end
```

**Note**: "This method is invoked many times during a page rendering and it triggers many queries (which will be cached, but we can save that with memoization)"

**Why it matters**: Even query cache has overhead - better to call once.

---


### Template Caching Strategy

**Pattern**: Layer caching at multiple levels - HTTP, template fragments, queries.

**Timeline caching**:

```ruby
# Controller - HTTP caching
class EventsController
  def index
    fresh_when @day_timeline
  end
end

# View - shared across users
<% cache [ user, filter, day.to_date, events ], "day-timeline" do %>
  <%= render "columns" %>
<% end %>

# Partial - filter menu cached per user
<% cache [ user, filter, expanded? ], "user-filtering" do %>
  <%= render "filters/menu" %>
<% end %>
```

> "I cached the filter menu since it implies rendering a bunch of templates and triggering many queries. The cache won't be shared across users, but it will be reused essentially in every rendered screen on a per-user basis."

**Caching layers**:
1. **HTTP cache** - Full response (via `fresh_when`)
2. **Column cache** - Shared across users with same events
3. **Filter menu** - Per-user, reused across pages
4. **Timezone** - Added to etag for user-specific rendering

**Why it matters**:
- Identify what varies (user, data, time)
- Cache at the right granularity
- Reuse fragments across requests

---

## Testing Patterns

### VCR for External APIs

**Pattern**: Record real API responses, replay in tests.

**AI translation tests**:

```ruby
# test/test_helper.rb
VCR.configure do |config|
  config.cassette_library_dir = "test/vcr_cassettes"
  config.hook_into :webmock
  config.filter_sensitive_data('<API_KEY>') { Rails.application.credentials.openai.api_key }
end

# test/models/command/ai/translator_test.rb
class Command::Ai::TranslatorTest < ActiveSupport::TestCase
  test "filter by assignments" do
    VCR.use_cassette("translator/filter_by_assignments") do
      result = Command::Ai::Translator.new("cards assigned to jz").translate
      assert_equal "jz", result.context[:assignees].first
    end
  end
end
```

**To update cassettes**: `VCR_RECORD=1 bin/rails test`

**Why it matters**:
- Fast tests (no network calls)
- Deterministic (same response every time)
- Works offline
- Documents actual API responses

---


### Development Tests OK

**Pattern**: It's fine to commit WIP tests, clean up before merge.

```ruby
# test/models/command/chat_query_test.rb
# > "This is a test I created for development purposes, I'll nix and create a proper suite"
```

**Why it matters**: Tests evolve with the feature; exploration tests help thinking.

---

## Rails Patterns

### Data.define for Value Objects

**Pattern**: Use Ruby 3.2's `Data.define` for immutable value objects.

```ruby
class Ai::Quota::Money < Data.define(:value)
  MICROCENTS_PER_DOLLAR = 100 * 1_000_000

  def self.wrap(value)
    new(convert(value))
  end

  def in_dollars
    value.to_d / MICROCENTS_PER_DOLLAR
  end
end

# Usage
money = Money.wrap("$100")
money.value        # => 10000000000 (microcents)
money.in_dollars   # => 100.0
```

**Benefits over Struct**:
- Immutable by default
- Pattern matching support
- Cleaner syntax

---


### Fixed-Point Arithmetic for Money

**Pattern**: Store money as integers (microcents) to avoid float errors.

**Discussion**:

```ruby
# Why not DECIMAL column?
# SQLite's DECIMAL is backed by float:
sqlite> SELECT (0.1 + 0.1 + 0.1) - 0.3 as difference;
5.55111512312578e-17

# Solution: INTEGER column with fixed-point conversion
class Ai::Quota::Money
  CENTS_PER_DOLLAR = 100
  MICROCENTS_PER_CENT = 1_000_000
  MICROCENTS_PER_DOLLAR = CENTS_PER_DOLLAR * MICROCENTS_PER_CENT

  def self.convert_dollars_to_microcents(dollars)
    (dollars.to_d * MICROCENTS_PER_DOLLAR).round.to_i
  end
end
```

**Why microcents, not cents?**
- LLM costs are fractions of a cent ($0.00001 per token)
- Microcents give 6 decimal places of precision

---


### Time-Based Reset Without Cron

**Pattern**: Check and reset on use, not scheduled job.

```ruby
class Ai::Quota
  def spend(cost)
    transaction do
      reset_if_due
      increment!(:used, cost.in_microcents)
    end
  end

  private
    def reset_if_due
      reset if due_for_reset?
    end

    def due_for_reset?
      reset_at.before?(Time.current)
    end

    def reset
      update(used: 0, reset_at: 7.days.from_now)
    end
end
```

**Why it matters**:
- One less moving part
- Resets happen exactly when needed
- No scheduler dependency

---


### Error Handling: Specific Errors

**Pattern**: Define custom errors for business rules.

```ruby
class Ai::Quota
  class UsageExceedsQuotaError < StandardError; end

  def ensure_not_depleted
    reset_if_due
    raise UsageExceedsQuotaError if depleted?
  end
end

# Controller
rescue_from Ai::Quota::UsageExceedsQuotaError do
  render json: { error: "You've depleted your quota" },
         status: :too_many_requests
end
```

**Why it matters**:
- Specific rescue blocks
- Clear intent
- HTTP status codes match business rules

---


### JavaScript Error Handling

**Pattern**: Parse error responses, show descriptive messages.

```javascript
// conversation/composer_controller.js
async #failPendingMessage(clientMessageId, response) {
  let errorMessage = null

  if (response?.contentType?.includes('application/json')) {
    const error = JSON.parse(await response.responseText)
    errorMessage = error.error
  }

  this.conversationMessagesOutlet.failPendingMessage(
    clientMessageId,
    errorMessage
  )
}
```

```css
/* Show error as data attribute */
.conversation__message--failed[data-error]::after {
  color: var(--color-negative);
  content: attr(data-error);
  display: block;
}
```

**Why it matters**: Users see "You've depleted your quota" not generic "Error occurred".

---

## Decision-Making Process

### Extract Only When Justified

**Progression**:
1. "The limit param is a smell that something is missing"
2. "Thinking about how to clarify this, my mind goes to having a new record"
3. "This way you remove the responsibility of tracking AI quotas from conversation"

**Teaching moment**: Shows *why* extraction helps, not just *what* to extract.

---


### Reconsider Based on New Information

**Money type decision**:
- First: "Could we clarify this with an object?"
- Then: "You could use a custom Active Model type"
- Finally: "I think it's good as it is... the current approach is simple enough"

**Why it matters**:
- No ego in code review
- Models learning process
- Best solution emerges through discussion

---


### The "Few Lines of Code" Heuristic

**Pattern**: Default to fewer lines unless more are clearly justified.

> "My rule of thumb is that the fewer lines of code the better, but of course, it's a fine line without absolute truths (sometimes the extra lines are justified). You have a much better sense of the problems with amounts here so follow your instinct 👍"

---

## Key Takeaways

1. **Narrow public APIs** - Only expose what's actually used
2. **Domain names over technical** - `depleted?` not `over_limit?`
3. **Objects emerge from coupling** - Shared params → extract object
4. **Memoize hot paths** - Methods called during rendering
5. **Layer caching** - HTTP, templates, queries
6. **Fixed-point for money** - Integers, not floats
7. **Reset on use, not cron** - Simpler, more reliable
8. **VCR for APIs** - Fast, deterministic tests
9. **Custom types: only when spread** - If used in one place, value object is enough
10. **Teach through questions** - "What do you think of..." not "Change this to..."

---

# Additional Rails Patterns

## Delegated Types for Polymorphism

Use `delegated_type` instead of traditional polymorphic associations:

```ruby
class Message < ApplicationRecord
  belongs_to :bubble, touch: true
  delegated_type :messageable, types: %w[Comment EventSummary],
                 inverse_of: :message, dependent: :destroy
end

module Messageable
  extend ActiveSupport::Concern
  included do
    has_one :message, as: :messageable, touch: true
  end
end
```

**Why**: Automatic convenience methods (`message.comment?`, `message.comment`) without manual type checking.

## Store Accessor for JSON Columns

Use `store_accessor` for structured JSON storage:

```ruby
class Bucket::View < ApplicationRecord
  store_accessor :filters, :order_by, :status, :assignee_ids, :tag_ids

  validates :order_by, inclusion: { in: ORDERS.keys, allow_nil: true }
end
```

**Why**: Type casting, validation, and cleaner API (`view.order_by` vs `view.filters['order_by']`).

## Normalizes for Data Consistency

Use `normalizes` to clean data before validation (Rails 7.1+):

```ruby
class Webhook < ApplicationRecord
  serialize :subscribed_actions, type: Array, coder: JSON

  normalizes :subscribed_actions,
    with: ->(value) { Array.wrap(value).map(&:to_s).uniq & PERMITTED_ACTIONS }
end
```

**Why**: Ensures data consistency before validation, no `before_validation` callbacks needed.

## Concern Organization by Responsibility

Split models into focused concerns:

```ruby
class Bubble < ApplicationRecord
  include Assignable      # Assignment logic
  include Boostable       # Boost counting
  include Eventable       # Event tracking
  include Poppable        # Archive logic
  include Searchable      # Full-text search
  include Staged          # Workflow stage logic
  include Taggable        # Tag associations
end
```

**Guidelines**:
- Each concern should be 50-150 lines
- Must be cohesive (related functionality together)
- Don't create concerns just to reduce file size

## Scopes Named for Business Concepts

```ruby
# Good - business-focused
scope :active, -> { where.missing(:pop) }
scope :unassigned, -> { where.missing(:assignments) }

# Not - SQL-ish
scope :without_pop, -> { ... }
scope :no_assignments, -> { ... }
```

## Transaction Wrapping

Wrap related updates for consistency:

```ruby
def toggle_stage(stage)
  transaction do
    update! stage: new_stage
    track_event event, stage_id: stage.id
  end
end
```

**When to use**: Multi-step operations, parent + children records, state transitions.

## Touch Chains for Cache Invalidation

```ruby
class Comment < ApplicationRecord
  has_one :message, as: :messageable, touch: true
end

class Message < ApplicationRecord
  belongs_to :bubble, touch: true
end
```

Changes propagate up: comment → message → bubble, invalidating caches automatically.

---

# What They Deliberately Avoid

> Patterns and gems 37signals chooses NOT to use.

---

## Notable Absences

The codebase is interesting as much for what's missing as what's present.

## Authentication: No Devise

**Instead**: ~150 lines of custom passwordless magic link code.

**Why avoid Devise**:
- Too heavyweight for passwordless auth
- Comes with password complexity they don't need
- Custom code is simpler to understand and modify

See [authentication.md](authentication.md) for the pattern.

## Authorization: No Pundit/CanCanCan

**Instead**: Simple predicate methods on models.

```ruby
# No policy objects - just model methods
class Card < ApplicationRecord
  def editable_by?(user)
    !closed? && (creator == user || user.admin?)
  end

  def deletable_by?(user)
    user.admin? || creator == user
  end
end

# In controller
def edit
  head :forbidden unless @card.editable_by?(Current.user)
end
```

**Why avoid authorization gems**:
- Simple predicates are easier to understand
- No separate policy files to maintain
- Logic lives with the model it protects

## Service Objects

**Instead**: Rich domain models with focused methods.

```ruby
# Bad - service object
class CardCloser
  def initialize(card, user)
    @card = card
    @user = user
  end

  def call
    @card.update!(closed: true, closed_by: @user)
    NotifyWatchersJob.perform_later(@card)
    @card
  end
end

# Good - model method
class Card < ApplicationRecord
  def close(by:)
    transaction do
      create_closure!(creator: by)
      notify_watchers_later
    end
  end
end
```

**Why avoid service objects**:
- They fragment domain logic across files
- Models become anemic (just data, no behavior)
- Simple operations don't need coordination objects

## Form Objects

**Instead**: Strong parameters and model validations.

```ruby
# No form objects - just params.expect
def create
  @card = @board.cards.create!(card_params)
end

private
  def card_params
    params.expect(card: [:title, :description, { tag_ids: [] }])
  end
```

**When form objects might be justified**: Complex multi-model forms. But even then, consider if nested attributes suffice.

## Decorators/Presenters

**Instead**: View helpers and partials.

```ruby
# No decorator gems
# Just helpers for view logic
module CardsHelper
  def card_status_badge(card)
    if card.closed?
      tag.span "Closed", class: "badge badge--closed"
    elsif card.overdue?
      tag.span "Overdue", class: "badge badge--warning"
    end
  end
end
```

## ViewComponent

**Instead**: ERB partials with locals.

```erb
<%# No ViewComponent - just partials %>
<%= render "cards/preview", card: @card, draggable: true %>
```

**Why partials are enough**:
- Simpler mental model
- No component class overhead
- Rails has good partial caching built-in

## GraphQL

**Instead**: REST endpoints with Turbo.

**Why avoid GraphQL**:
- Adds complexity for uncertain benefit
- REST + Turbo handles their needs
- No mobile app requiring flexible queries

## Sidekiq

**Instead**: Solid Queue (database-backed).

**Why avoid Sidekiq**:
- Removes Redis dependency
- Database is already managed
- Good enough for their scale

## React/Vue/Frontend Framework

**Instead**: Turbo + Stimulus + server-rendered HTML.

**Why avoid SPAs**:
- Server rendering is simpler
- Less JavaScript to maintain
- Turbo provides SPA-like feel
- Stimulus handles interactions

## Tailwind CSS

**Instead**: Native CSS with cascade layers.

**Why avoid Tailwind**:
- Native CSS has nesting, variables, layers now
- No build step complexity
- Semantic class names preferred

## RSpec

**Instead**: Minitest (ships with Rails).

**Why avoid RSpec**:
- Minitest is simpler, less DSL
- Faster boot time
- Good enough assertions

## FactoryBot

**Instead**: Fixtures.

**Why avoid factories**:
- Fixtures are faster (loaded once)
- Relationships are explicit in YAML
- Deterministic test data

## The Philosophy

> "We reach for gems when Rails doesn't provide a solution. But Rails provides most solutions."

Before adding a dependency, ask:
1. Can vanilla Rails do this?
2. Is the complexity worth the benefit?
3. Will we need to maintain this dependency?
4. Does it make the codebase harder to understand?

## What They DO Use

Some gems that made the cut:

- `solid_queue`, `solid_cache`, `solid_cable` - Database-backed infrastructure
- `turbo-rails`, `stimulus-rails` - Hotwire
- `propshaft` - Simple asset pipeline
- `kamal` - Deployment
- `bcrypt` - Password hashing (for magic link tokens)
- `image_processing` - Active Storage variants

The bar is high. Each gem must clearly earn its place.

```