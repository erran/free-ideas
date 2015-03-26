# RSpec matcher's diffable through attribute
## Proposal
Currently defining a diffable matcher will perform a diff on the actual and expected values.

I propose that we support sending attributes as the "actual" and "expected" values used in the diffing process.

### Example
```ruby
# matchers.rb
RSpec::Matchers.define_matcher :a_criterion_matching do |expected|
  match do |actual|
    next false unless expected[:field].eql?(actual.field)
    next false unless expected[:operator].eql?(actual.operator)
    next false unless expected[:value].eql?(actual.value)

    true
  end

  diffable through_attributes: [:field, :operator, :value]
end

# *_spec.rb
criterion1 = Criterion.new(field: 'BAR', operator: 'CONTAINS', value: 'test')
criteria = Criteria.new([criterion1])
expect(criteria.criteria).to include(
  a_criterion_matching(field: 'FOO', operator: 'IS', value: 'test')
)
```

```diff
@@ -1,5 +1,5 @@
 {
-  :field => 'FOO',
-  :operator => 'IS',
+  :field => 'BAR',
+  :operator => 'CONTAINS',
   :value => 'test'
 }
```

## Background
I found the following code in the wild at work.
```ruby
site_criteria = Nexpose::Site.load(@console, site_id).search_criteria
expect(site_criteria.criteria.first.field).to eq('CLUSTER')
expect(site_criteria.criteria.first.operator).to eq('IS')
expect(site_criteria.criteria.first.value.first).to eq('test')
```

It really bugged me for a few reasons:

1. We're making three separate assertions when all we really cared about was validating that at least one item existed with matching attributes.
2. We have several version of it across out test suite.
3. The `site_criteria.criteria.first` call is made several times.

I initially decided to create a custom matcher similiar to the built-in `have_attributes` matcher.
```ruby
site_criteria = Nexpose::Site.load(@console, site_id).search_criteria
expect(site_criteria).to include_criterion(field: 'MAC_ADDRESS', operator: 'CONTAINS', value: '456')
```

This custom matcher was not diffable since it wasn't as simple as calling `diffable` when dealing with nested attributes.

I realized I could alias `have_attributes` to `a_criterion_matching` and use the `include` composable matcher:
```ruby
RSpec::Matchers.alias_matcher :a_criterion_matching, :have_attributes

site_criteria = Nexpose::Site.load(@console, site_id).search_criteria
expect(site_criteria.criteria).to include(
  a_criterion_matching(field: 'MAC_ADDRESS', operator: 'CONTAINS', value: '456')
)
```

While this is the most idomatic in terms of RSpec usage the diff could still be improved:

```diff
@@ -1,2 +1,5 @@
-[(a criterion matching {:field => "MAC_ADDRESS", :operator => "CONTAINS", :value => "456"})]
+[#<Nexpose::Criterion:0x007fced6ecc618
+  @field="MAC_ADDRESS",
+  @operator="CONTAINS",
+  @value="123">]
```

You'll notice that the diff is between the matcher and the inspected object.
Ideally the diff would only show that the only actual difference is the "value" attribute.
