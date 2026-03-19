# SPL Command Reference

## Official Documentation Links

### Search Language
- [SPL Search Reference](https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.2/search-commands/about-the-search-reference)
- [SPL2 Search Reference](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/introduction/introduction)
- [SPL2 Eval Functions](https://help.splunk.com/en/splunk-enterprise/search/spl2-search-reference/evaluation-functions/quick-reference-for-spl2-eval-functions)

### Key Commands
- [eval](https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.2/search-commands/eval)
- [stats](https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.2/search-commands/stats)
- [tstats](https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.2/search-commands/tstats)
- [rex](https://help.splunk.com/en/splunk-enterprise/search/spl-search-reference/10.2/search-commands/rex)

### Search Optimization
- [About Search Optimization](https://help.splunk.com/en/splunk-enterprise/search/search-manual/9.4/optimizing-searches/about-search-optimization)
- [Quick Tips for Optimization](https://help.splunk.com/en/splunk-enterprise/search/search-manual/9.2/optimizing-searches/quick-tips-for-optimization)
- [Write Better Searches](https://help.splunk.com/en/splunk-enterprise/search/search-manual/9.4/optimizing-searches/write-better-searches)

### Data Models & Knowledge
- [Common Information Model (CIM)](https://help.splunk.com/en/data-management/common-information-model/6.3)
- [Splunk Regular Expressions](https://help.splunk.com/en/splunk-enterprise/search/knowledge-manager/about-splunk-regular-expressions)

## Quick Reference

### Command Categories
**Streaming** (fast): eval, where, rename, fields, rex, spath, lookup, fillnull, replace, regex, search

**Transforming** (place late): stats, chart, timechart, top, rare, sort, dedup, head, tail, table, transpose

**Generating**: tstats, inputlookup, makeresults, rest, datamodel, metadata

**Dataset**: append, join, union, appendpipe

### Common Regex Patterns
```
IP:    \d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}
Email: [\w.+-]+@[\w-]+\.[\w.]+
URL:   https?://[^\s]+
MAC:   [0-9a-fA-F]{2}(:[0-9a-fA-F]{2}){5}
UUID:  [0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}
```

### TERM/PREFIX Directives
```spl
TERM(192.168.1.100)      # IP as single indexed term
PREFIX(api/v2/)           # Match term beginnings
```

### Optimization Checklist
1. Specify index, sourcetype, source
2. Use field-value pairs before first pipe
3. Filter before aggregation
4. Use `fields` to limit extraction
5. Avoid wildcards and NOT
6. Use TERM() for minor-breaker strings
7. Use tstats + accelerated data models
8. Set appropriate time ranges
9. Use summary indexing for expensive recurring searches
