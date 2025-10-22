# HubSpot CRM Objects Batch Fetcher

## Overview

This HubSpot HubL template provides a reusable pattern for retrieving large datasets from custom CRM objects using batch processing and pagination. It efficiently handles API limitations by fetching data in chunks and automatically stopping when all records are retrieved.

## Features

- **Batch Processing**: Fetches data in configurable chunks (default: 100 records per call)
- **Automatic Pagination**: Handles multiple pages of results seamlessly
- **Smart Termination**: Stops making API calls when all records are retrieved
- **API Limit Compliance**: Respects HubSpot's 10 API calls per page limit
- **Flexible Configuration**: Easy to customize for different CRM objects and queries

## Template Structure

```jinja2
{% set array_of_objects = [] %}
{% set batch_size = 100 %}
{% set max_calls = 5 %} {# max number of times to call crm_objects is 10 per page #}

{% for i in range(0, max_calls) %}
  {% set pagination = "&limit=" ~ batch_size ~ "&offset=" ~ i * batch_size %}
  {% set fqn = "custom_object_fqn" %}
  {% set query = "query string here" ~ pagination %}
  {% set fields = "field1,field2,field3" %}
  {% set format = true/false %}
  {% set batch = crm_objects( fqn, query, fields, format ) %}
  {% do array_of_objects.extend(batch.results) %}
  
  {% if not batch.has_more %}
    {% break %}
  {% endif %}
{% endfor %}

{{ array_of_objects }}
```

## Configuration Parameters

### Variables

| Variable | Description | Default | Notes |
|----------|-------------|---------|-------|
| `array_of_objects` | Array to store all fetched records | `[]` | Accumulates results from all batches |
| `batch_size` | Number of records per API call | `100` | Max recommended: 100 |
| `max_calls` | Maximum number of API calls | `5` | HubSpot limit: 10 per page |

### crm_objects() Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `fqn` | String | Fully qualified name of the custom object | `"p_club_champions"` |
| `query` | String | Query string with filters and parameters | `"year=2024&limit=100&offset=0"` |
| `fields` | String | Comma-separated list of fields to retrieve | `"name,email,phone"` |
| `format` | Boolean | Whether to format the response | `true` or `false` |

## Usage Examples

### Example 1: Fetch Club Champions by Year

```jinja2
{% set all_champions = [] %}
{% set batch_size = 100 %}
{% set max_calls = 5 %}

{% for i in range(0, max_calls) %}
  {% set pagination = "&limit=" ~ batch_size ~ "&offset=" ~ i * batch_size %}
  {% set fqn = "p_club_champions" %}
  {% set query = "year=" ~ year ~ "&club_name__not_null=true&order=club_name" ~ pagination %}
  {% set fields = "club_name,junior_boy_s,junior_girl_s,men_s,senior_men_s,senior_women_s,women_s,year" %}
  {% set format = false %}
  {% set batch = crm_objects(fqn, query, fields, format) %}
  {% do all_champions.extend(batch.results) %}
  
  {% if not batch.has_more %}
    {% break %}
  {% endif %}
{% endfor %}

{# Display results #}
{% for champion in all_champions %}
  <div class="champion-card">
    <h3>{{ champion.club_name }}</h3>
    <p><strong>Men's Champion:</strong> {{ champion.men_s }}</p>
    <p><strong>Women's Champion:</strong> {{ champion.women_s }}</p>
  </div>
{% endfor %}
```

### Example 2: Fetch Active Members with Email

```jinja2
{% set all_members = [] %}
{% set batch_size = 100 %}
{% set max_calls = 10 %}

{% for i in range(0, max_calls) %}
  {% set pagination = "&limit=" ~ batch_size ~ "&offset=" ~ i * batch_size %}
  {% set fqn = "p_members" %}
  {% set query = "status=active&email__not_null=true&order=last_name" ~ pagination %}
  {% set fields = "first_name,last_name,email,phone,membership_type" %}
  {% set format = true %}
  {% set batch = crm_objects(fqn, query, fields, format) %}
  {% do all_members.extend(batch.results) %}
  
  {% if not batch.has_more %}
    {% break %}
  {% endif %}
{% endfor %}

<p>Total Active Members: {{ all_members|length }}</p>
```

### Example 3: Fetch Recent Registrations

```jinja2
{% set recent_registrations = [] %}
{% set batch_size = 50 %}
{% set max_calls = 3 %}

{% for i in range(0, max_calls) %}
  {% set pagination = "&limit=" ~ batch_size ~ "&offset=" ~ i * batch_size %}
  {% set fqn = "p_event_registrations" %}
  {% set query = "event_date__gte=" ~ start_date ~ "&order=-created_at" ~ pagination %}
  {% set fields = "event_name,attendee_name,registration_date,ticket_type" %}
  {% set format = false %}
  {% set batch = crm_objects(fqn, query, fields, format) %}
  {% do recent_registrations.extend(batch.results) %}
  
  {% if not batch.has_more %}
    {% break %}
  {% endif %}
{% endfor %}
```

## Query String Parameters

### Common Query Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `order` | Sort field (use `-` for descending) | `order=-created_at` |
| `fieldname__not_null` | Filter for non-null values | `email__not_null=true` |
| `fieldname__eq` | Exact match filter | `status__eq=active` |
| `fieldname__gte` | Greater than or equal to | `date__gte=2024-01-01` |
| `fieldname__lte` | Less than or equal to | `date__lte=2024-12-31` |

**Note**: `limit` and `offset` parameters are automatically added via the `pagination` variable and should not be included in your base query string.

## How Pagination Works

The template implements offset-based pagination:

1. **First call** (`i=0`): `offset=0` → Records 1-100
2. **Second call** (`i=1`): `offset=100` → Records 101-200
3. **Third call** (`i=2`): `offset=200` → Records 201-300
4. **...and so on**

The loop automatically terminates when `batch.has_more` returns `false`, indicating no more records exist.

## Performance Considerations

### Optimal Settings

- **Batch Size**: 100 records (recommended maximum)
- **Max Calls**: Start with 5, increase only if needed
- **Total Records**: `batch_size × max_calls` = maximum records retrieved

### API Limits

⚠️ **Important**: HubSpot limits `crm_objects()` to **10 calls per page load**

```jinja2
{% set max_calls = 10 %}  {# Maximum allowed #}
```

### Best Practices

1. **Use filters** to reduce dataset size before fetching
2. **Request only needed fields** to minimize response size
3. **Set appropriate max_calls** based on expected data volume
4. **Cache results** when possible to avoid repeated API calls
5. **Monitor performance** and adjust batch size if needed

## Troubleshooting

### Issue: Not All Records Retrieved

**Problem**: Data seems incomplete

**Solutions**:
- Increase `max_calls` (up to 10)
- Check if filters are too restrictive
- Verify the query syntax is correct

### Issue: Page Load Timeout

**Problem**: Page takes too long to load

**Solutions**:
- Reduce `max_calls`
- Decrease `batch_size`
- Add more specific filters to reduce total records
- Consider caching strategy

### Issue: Empty Results

**Problem**: `array_of_objects` is empty

**Solutions**:
- Verify the custom object FQN is correct
- Check filter conditions in query string
- Ensure fields exist in the custom object
- Test query with `format=true` for debugging

## Advanced Usage

### Counting Total Records

```jinja2
{% set total_count = all_records|length %}
<p>Retrieved {{ total_count }} total records</p>
```

### Filtering Results After Fetch

```jinja2
{% set filtered = [] %}
{% for record in all_records %}
  {% if record.some_field == "some_value" %}
    {% do filtered.append(record) %}
  {% endif %}
{% endfor %}
```

### Grouping by Field

```jinja2
{% set grouped = {} %}
{% for record in all_records %}
  {% set key = record.category %}
  {% if key not in grouped %}
    {% do grouped.update({key: []}) %}
  {% endif %}
  {% do grouped[key].append(record) %}
{% endfor %}
```

## Response Format

### With `format=false`

```json
{
  "field1": "value1",
  "field2": "value2",
  "field3": "value3"
}
```

### With `format=true`

Returns formatted values suitable for display (dates formatted, numbers with proper separators, etc.)

## Security Considerations

- Never expose sensitive fields in public templates
- Use appropriate permissions on custom objects
- Validate and sanitize any user inputs used in queries
- Be cautious with `format=false` as it may expose raw data

## Resources

- [HubSpot HubL Documentation](https://developers.hubspot.com/docs/cms/hubl)
- [CRM Objects in HubL](https://developers.hubspot.com/docs/cms/hubl/functions#crm-objects)
- [Custom Objects API](https://developers.hubspot.com/docs/api/crm/crm-custom-objects)

## License

This template pattern is provided as-is for use with HubSpot CMS.

## Contributing

Feel free to adapt this pattern for your specific use cases. Common modifications include:
- Adding error handling
- Implementing caching mechanisms
- Creating reusable macros
- Adding data transformation logic

---

**Note**: Always test thoroughly in a development environment before deploying to production.
