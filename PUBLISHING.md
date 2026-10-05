# Publishing to Odoo Apps

Module: `nx_audit_log` — Odoo 18.0. Free of charge; existing OPL-1 license retained.

1. Test installation and the existing module tests on a disposable Odoo 18 database before publishing:

   `odoo-bin -d audit_test -i nx_audit_log --test-enable --test-tags nx_audit --stop-after-init`

2. Create an empty GitHub repository named `odoo_apps` (do not initialize a README).
3. In this directory, connect and push it, replacing YOUR_USERNAME:

   ```sh
   git remote add origin https://github.com/YOUR_USERNAME/odoo_apps.git
   git push -u origin 18.0
   ```

4. If private, grant repository access to GitHub user `online-odoo`.
5. Sign in at https://apps.odoo.com/apps/upload and register:

   `ssh://git@github.com/YOUR_USERNAME/odoo_apps#18.0`

6. Review the scan results and the rendered app page.

References:
- https://apps.odoo.com/apps/upload
- https://apps.odoo.com/apps/vendor-guidelines

Local preparation checks Python/XML syntax and referenced files only. It does not replace installation, behavior, security, or compatibility testing.
