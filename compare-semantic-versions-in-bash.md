How to compare semantic versions strings in bash using python
=============================================================

This isn't pure bash, but the pure bash implementations I've found are pretty
long and ugly.  This solution gives me the balance I'm looking for between pure
bash and readability.

I use the following shell function.  My version strings are of the form
`release-X.Y` or `master`.  `master` should always be the latest version.

    compare_versions() {
        local aver="$1"
        local op="$2"
        local bver="$3"
        if [ "$aver" = master ] ; then aver=release-9999 ; fi
        if [ "$bver" = master ] ; then bver=release-9999 ; fi
        python -c 'import sys
    from pkg_resources import parse_version
    sys.exit(not parse_version(sys.argv[1])'"${op}"'parse_version(sys.argv[2]))' "$aver" "$bver"
    }

For example, if the version is `release-3.11` and I want to take some action if
the release is `release-3.10` or later:

    if compare_versions $version '>=' release-3.10 ; then
        echo do something for newer versions
    else
        echo do something else for older versions
    fi

If `$version` is `release-3.10`, `release-3.11`, or `master` it will hit the
`echo do something for newer versions` branch, otherwise it will hit the other
branch.
