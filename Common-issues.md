* Verify that your JSON configuration files are syntactically correct before invoking GenomicsDB tools

    The GenomicsDB tools assume that you provide a syntactically correct JSON - always validate your JSON files using 
tools such as _json_verify_ before invoking GenomicsDB tools.

        cat <json_file> | json_verify

* You cannot move/copy the TileDB workspace/array directory arbitrarily after loading. If you do copy/move files to a different machine, ensure that the absolute path of the workspace/array directory is the same as that on the machine on which data is loaded/imported.
