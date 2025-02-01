#!/usr/bin/perl -w
#
# this script generates output for input to nsupdate
#  making it easy to update dynamic dns
#
# accepts input list of hostnames with IPv4 or IPv6 addresses
#  supported formats: special inventory, basic csv, host=ip
#   appends a domain suffix to hostnames, if needed
#    generating both forward and reverse updates
#
# requires the Net::IP module
#
# created June 2008 by Robert Patrick;
#  updated December 10, 2010; added loop to handle multiple input lines
#  updated July 28, 2012; now supports both basic csv plus inventory format
#
### This is free and unencumbered software released into the public domain.
###
### Anyone is free to copy, modify, publish, use, compile, sell, or
### distribute this software, either in source code form or as a compiled
### binary, for any purpose, commercial or non-commercial, and by any
### means.
###
### In jurisdictions that recognize copyright laws, the author or authors
### of this software dedicate any and all copyright interest in the
### software to the public domain. We make this dedication for the benefit
### of the public at large and to the detriment of our heirs and
### successors. We intend this dedication to be an overt act of
### relinquishment in perpetuity of all present and future rights to this
### software under copyright law.
###
### THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
### EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
### MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
### IN NO EVENT SHALL THE AUTHORS BE LIABLE FOR ANY CLAIM, DAMAGES OR
### OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE,
### ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
### OTHER DEALINGS IN THE SOFTWARE.
#
#
# Example usage:
#
# echo "site,bldg,hostname,1.2.3.4,other.info" | mkdns | nsupdate
#
# echo "hostname,1.2.3.4" | mkdns -nsupdate
#
# mkdns < list | nsupdate
#
# mkdns -ns < list
#
# mkdns -ns -host "foo=1.2.3.4" -host "bar=2.3.4.5"
#
# mkdns -ns "foo=1.2.3.4" "bar,2.3.4.5"
#
# rmdns -ns "foo=1.2.3.4"

 use strict;
 use Net::IP;
 our $VERSION = "2012.07.29";

 my $dns_server = 'authoriative-dns-server-name-or-ip'; # send dynamic dns updates to this server

 my $domain = 'example.com'; # default domain suffix if none present for hostnames

 my $nsupdate = '/usr/bin/nsupdate -v -t 30'; # location and args for nsupdate
 my $use_nsupdate = 0; # set to 1 will pipe script output to nsupdate by default

 my $debug = 0; # set to 1 generates standard output, set higher for more output

 my $testmode = 0; # by default, we don't run in test mode

 my $drop_suffix = 0; # set to 1 if you want to discard any input domain suffixes
                      # this option forces the use of $domain for all records

 my $remove_records = 0; # set remove mode disabled by default unless called as rmdns
 $remove_records = 1 if ($0 =~ m/rmdns/); # when called as rmdns, we delete records

 my $do_ptr = 1;    # by default, modify ptr records
 my $do_a   = 1;    # by default, modify host records (A or AAAA)

 my $del_any_a = 1; # by default, first delete any existing A or AAAA record and then
                    # recreate it - this ensures forward lookups resolve to one address
                    # disabling allows multiple addresses per hostname
                    # if set 0, append instead of replace addresses per hostname

 my (%hosts, $list, $hostname, $ip, $reverse_ip, $ip_version);

 my $counter = 0;
 my $ttl = 3600;

 my $scriptname = $0;
 $scriptname =~ s/.*\///g;

 use Getopt::Long;
 Getopt::Long::Configure("pass_through");
 GetOptions(
        'd|debug+' => \$debug,
        'domain=s' => \$domain,
        'dd|dropdomain!' => \$drop_suffix,
        'server=s' => \$dns_server,
        'remove!' => \$remove_records,
        'ptr!' => \$do_ptr,
        'a|aaaa!' => \$do_a,
        'deletea!' => \$del_any_a,
        'n|ns|nsupdate!' => \$use_nsupdate,
        'testmode!' => \$testmode,
        'host=s%' => sub { push(@{$hosts{$_[1]}}, $_[2]) },
        'h|help|?' => sub { version(); usage(); exit; },
        'version' => sub { version(); exit; },
        );

 if ($debug >=3) {
        # display command line options if multi debug enabled
        print "*** begin show cli input ***\n";
        print " debug       = $debug\n";
        print " test mode   = $testmode\n";
        print " nsupdate    = $use_nsupdate\n";
        print " server      = $dns_server\n";
        print " domain      = $domain\n";
        print " drop domain = $drop_suffix\n";
        print " remove mode = $remove_records\n";
        print " delete a    = $del_any_a\n";
        print " do ptr      = $do_ptr\n";
        print " do a        = $do_a\n";
        foreach (@ARGV) {
                print " extra: $_\n";
        }
        print "*** end show cli input ***\n";
 }

 if ($use_nsupdate) {
        my ($exec, $args) = split(/ /,$nsupdate,2);
        if ((-e $exec) and (-x $exec) and (! -d $exec)) {
                # if nsupdate program file exists, is executable, and not a directory, we can continue
                warn("\n*** calling nsupdate directly, do not pipe this output to nsupdate manually ***\n\n") if ($debug);
                open(my $fh, "| $nsupdate") or die "failed to open nsupdate! $!";
                $use_nsupdate = $fh;
        } else {
                print STDERR "$scriptname: error accessing file ($nsupdate)\n";
                exit(255);
        }
 }
 output("server $dns_server\n");


## begin main loop
 if ((%hosts) or (@ARGV)) {
        # if given input directly on the command line
        foreach my $input (@ARGV) {
                process_input($input);
        }
        # if given hostnames as -host "x=y" options
        while ( ($hostname, $list) = each(%hosts) ) {
                foreach $ip (@$list) {
                        process_input("$hostname,$ip");
                }
        }
 } else {
        # else parse stdin looking for input
        while(<STDIN>)
        {
                process_input($_);
        }
}

## end main loop, clean-up and quit
 if ($use_nsupdate) {
        my $result;
        eval {
                close($use_nsupdate); # or die "failed to close nsupdate! $!";
                $result = $?;
        };
        print STDERR "$scriptname: error encounted while running nsupdate ($result)\n" if ($result);
 }
exit;


## subroutines follow

sub process_input {
 $_ = shift;
 $counter++;
 chomp;      # no newline
 s/^!.*//;   # no comment lines leading !
 s/^#.*//;   # no comment lines leading #
 s/#.*//;    # drop anything after #
 s/^\s+//;   # no leading white
 s/\s+$//;   # no trailing white
 s/\s+/ /g;  # reduce multiple spaces to one
 s/,\s/,/g;  # replace comma+space with comma
 s/\s,/,/g;  # replace space+comma with comma
 return if (! length); # anything left?
 if (! m/,/) {
        # if no comma present within the remaining input
        # replace any spaces still remaining with commas
        s/\s/,/g;
        s/=/,/; # also replace first equal = with comma
        s/\|/,/; # and replace first pipe | with comma
 }

 my $field_count = ( tr/,// );
 return unless ($field_count);

 my ($record_type, $site, $bldg, $mas);

 if (($field_count == 1) or ($field_count == 2)) {
        # one or two commas, try basic csv format
        ($hostname, $ip, $mas) = split(',',$_, 3);
 } elsif ($field_count >= 3) {
    # three or more commas, try inventory format
    ($site, $bldg, $hostname, $ip, $mas) = split(',',$_, 5);
 } else {
    print STDERR "$scriptname: illegal record at $counter\n";
    return;
 }

 ($hostname, $ip, $reverse_ip, $ip_version) = check_ip($hostname,$ip);
 unless($ip) {
        print STDERR "$scriptname: illegal ip address input at $counter\n";
        return;
 }

 if ($drop_suffix) {
        $hostname =~ s/\..*//; # drop any domain suffix
        $hostname = lc($hostname . '.' . $domain . '.'); # lowercase name and add suffix
 } else {
        $hostname = "$hostname.$domain" unless ($hostname =~ /\./); # add suffix if none present
        $hostname = lc($hostname); # ensure hostname is all lowercase
 }

 # set forward dns record type to A for v4, AAAA for v6
 ($ip_version eq 4) ? ($record_type = "a") : ($record_type = "aaaa");

 # output nsupdate syntax for atomic transactions
 if ($do_a) {
        output("update delete $hostname $record_type\n") if ($del_any_a);
        output("update add $hostname $ttl $record_type $ip\n") if (! $remove_records);
        output("show\n") if ( ($debug >= 2) or (($debug) and (! $use_nsupdate)));
        output("send\n");
 }
 if ($do_ptr) {
        output("update delete $reverse_ip\n");
        output("update add $reverse_ip $ttl ptr $hostname\n") if (! $remove_records);
        output("show\n") if (($debug >= 2) or (($debug) and (! $use_nsupdate)));
        output("send\n");
 }
 return;
}

sub output {
 my $string = shift;
 print "$string" if (($debug) or (! $use_nsupdate));
 print {$use_nsupdate} "$string" if (($use_nsupdate) and (! $testmode));
 return;
}

sub check_ip {
 my ($hostname,$ip_address) = @_;
 my ($ip_version, $reverse_ip);
 unless ($ip = new Net::IP($ip_address)) {
        # if input ip isn't an ip address
        # maybe the user swapped the input
        # as "ip,host" insead of "host,ip"
        $ip = new Net::IP($hostname);
        if (! $ip) {
                # nope, the hostname wasn't an ip
                # so blank out both to return null
                $ip_address = ''; $hostname = '';
        } else {
                # yes sir, the user swapped elements
                # so we adapt and overcome
                my $temp = $hostname;
                $hostname = $ip_address;
                $ip_address = $temp;
        }
 }
 if ($ip) {
        $ip_version = $ip->version();
        if (($ip_version eq '6') or (($ip_version eq '4') and ($ip_address =~ m/\b(?:\d{1,3}\.){3}\d{1,3}\b/))) {
                # run this check to confirm dotted quad is a full ip address, Net::IP allows abbreviations but we don't
                $ip_address = $ip->ip();
                $reverse_ip = $ip->reverse_ip();
        } else {
                $ip_address = ''; $hostname = '';
        }
 }
 return($hostname,$ip_address,$reverse_ip,$ip_version);
}

sub version {
 print "$scriptname version $VERSION\n";
 return;
}

sub usage {
 print <<EOF;
This software is especially flammable and comes with ABSOLUTELY NO WARRANTY.
You're likely to crash and burn, corrupting customer DNS data, so stop now!
Danger ahead.  Proceed at your own risk.

This script is designed for automating updates to Dynamic DNS records.

Usage:
$scriptname [OPTIONS] -host "host1=1.2.3.4" -host "host2=2001:db8::123"
$scriptname [OPTIONS] "host1=1.2.3.4" "host2=2001:db8::123" "host3,4.5.6.7"

  or pipe input via STDIN

$scriptname < input.file

Options
 -d, --debug        display increasingly verbose debug output
 -t, --test         enables test mode, negates output to nsupdate
 -n, --nsupdate     enables direct nsupdate access
 --server=SERVER    define server for DDNS update (default = $dns_server)
 --domain=DOMAIN    define default domain (default = $domain)

 -dd, --dropdomain  discards domain suffixes, forces default domain
 -noa               avoid updating A or AAAA forward records
 -noptr             avoid updating PTR reverse records

 -nodeletea         allows multiple addresses per A or AAAA record

 -remove            enable removal mode, deleting DDNS records

EOF
 return;
}

__END__
