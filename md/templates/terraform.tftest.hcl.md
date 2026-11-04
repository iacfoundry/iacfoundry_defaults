# --------------------------------------------------------------
# Terraform Module Tests
# Contributor(s): {developer}
# Purpose: Terraform Resource Module Test Provider and run steps
# --------------------------------------------------------------

{mock_providers}

run "mock_plan" {
  providers = {
    {provider_mappings}
  }
  command = plan
}
