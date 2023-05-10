from genie.testbed import load
from genie.utils.config import Config
from genie.metaparser.util.exceptions import SchemaEmptyParserError
from pyats import aetest
import yaml

class TacacsCompliance(aetest.Testcase):

    @aetest.setup
    def connect(self, testbed):
        # Load the testbed file and connect to the device
        self.device = testbed.devices['ios_device']
        self.device.connect()

    @aetest.test
    def check_tacacs_compliance(self):
        # Define the YAML template file path
        template_file = 'tacacs_template.yaml'

        # Execute the "show running-config" command and parse the output
        try:
            output = self.device.execute('show running-config | include ^tacacs')
        except Exception as e:
            self.failed("Unable to execute command: {}".format(e))

        # Parse the output using Genie
        try:
            parsed_output = self.device.parse('show running-config | include ^tacacs')
        except SchemaEmptyParserError:
            self.failed("Unable to parse output of 'show running-config'")

        # Load the YAML template file
        try:
            with open(template_file, 'r') as f:
                template = yaml.safe_load(f)
        except Exception as e:
            self.failed("Unable to load YAML template file: {}".format(e))

        # Compare the parsed output with the YAML template
        cfg = Config(parsed_output)
        compliant = cfg == Config(template)

        # Print the compliance status
        if compliant:
            self.passed("TACACS configuration is compliant.")
        else:
            self.failed("TACACS configuration is not compliant.")

    @aetest.cleanup
    def disconnect(self):
        # Disconnect from the device
        self.device.disconnect()

if __name__ == '__main__':
    # Define the testbed file
    testbed_file = 'testbed.yaml'

    # Create and run the test suite
    testbed = load(testbed_file)
    aetest.main(testbed=testbed)
