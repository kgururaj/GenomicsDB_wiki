* Verify that your JSON configuration files are syntactically correct before invoking GenomicsDB tools

    The GenomicsDB tools assume that you provide a syntactically correct JSON - always validate your JSON files using 
tools such as _json_verify_ before invoking GenomicsDB tools.

        cat <json_file> | json_verify
